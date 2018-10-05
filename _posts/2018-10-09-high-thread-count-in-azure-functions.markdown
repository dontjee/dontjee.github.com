---
layout: "post"
title: "High thread count in Azure Functions"
description: ''
category: null
tags: []
---

Recently Azure Functions runtime V2 was released and one of my team's functions started to consume an abnormally high number of threads. When we dug into the details of the problem, the root cause was performing static initialization of logging on each function invocation. Specifically, we were using Serilog with the Application Insights sink and were recreating the sink each time the function ran. The sink was then assigned to a static variable (`Log.Logger`).

The code used to look something like this:

```csharp
using System;
using Microsoft.ApplicationInsights.Extensibility;
using Microsoft.Azure.WebJobs;
using Serilog;

namespace ServiceBusThreadingProblem
{
   public static class HighThreadCountFunction
   {
      [FunctionName( "HighThreadCountFunction" )]
      public static void Run( [ServiceBusTrigger( "TestTopicName", "TestSubscriptionName", Connection = "TestSubscriptionConnectionString" )] string messageContents, ExecutionContext context, int deliveryCount, DateTime enqueuedTimeUtc, string messageId )
      {
         var logConfiguration = new LoggerConfiguration()
            .Enrich.WithProperty( "FunctionInvocationId", context.InvocationId )
            .Enrich.WithProperty( "QueueMessageMessageId", messageId )
            .Enrich.WithProperty( "MachineName", Environment.MachineName );

         logConfiguration.WriteTo.ApplicationInsightsTraces( new TelemetryConfiguration( Environment.GetEnvironmentVariable( "APPINSIGHTS_INSTRUMENTATIONKEY" ) ) );

         Log.Logger = logConfiguration.CreateLogger();
         Log.Information( $"C# ServiceBus topic trigger function processed message: {messageContents}" );
      }
   }
}
```

The issue is with the re-assignment of the `Log.Logger` static variable each time the function runs. Looking at a memory dump it was apparent that each thread (~400 of them) was waiting somewhere in Application Insights code. That led us to examine the application insights traces line, which is part of the Serilog initialization. A quick test confirmed that without that line, the threads stayed at normal levels.

As a general principle in Azure Functions, if a class can manage external connections and is threadsafe then it should be reused as a static variable. See the [Azure Function documentation on static client reuse](https://docs.microsoft.com/en-us/azure/azure-functions/manage-connections#use-static-clients). Since Serilog's `Log.Logger` is a static variable and is threadsafe, the code above should not have been recreating the logger on each invocation.

Instead, the code above should be refactored to reuse the `Log.Logger` variable. One pattern that makes this easy is using the `Lazy` class to ensure only one instance of the logger is ever created as shown below.

```csharp
using System;
using Microsoft.ApplicationInsights.Extensibility;
using Microsoft.Azure.WebJobs;
using Serilog;
using Serilog.Context;
 
namespace ServiceBusThreadingProblem
{
   public static class NormalThreadCountFunction
   {
      private static Lazy<ILogger> LoggingInitializer = new Lazy<ILogger>( () =>
      {
         var logConfiguration = new LoggerConfiguration()
                 .Enrich.FromLogContext()
                 .Enrich.WithProperty( "MachineName", Environment.MachineName );

         logConfiguration.WriteTo.ApplicationInsightsTraces( new TelemetryConfiguration( Environment.GetEnvironmentVariable( "APPINSIGHTS_INSTRUMENTATIONKEY" ) ) );
         Log.Logger = logConfiguration.CreateLogger();
         return Log.Logger;
      } );

      [FunctionName( "NormalThreadCountFunction" )]
      public static void Run( [ServiceBusTrigger( "TestTopicName", "TestSubscriptionName", Connection = "TestSubscriptionConnectionString" )] string messageContents, ExecutionContext context, int deliveryCount, DateTime enqueuedTimeUtc, string messageId )
      {
         ILogger logger = LoggingInitializer.Value;

         using ( LogContext.PushProperty( "FunctionInvocationId", context.InvocationId ) )
         using ( LogContext.PushProperty( "QueueMessageMessageId", messageId ) )
         {
            logger.Information( $"C# ServiceBus topic trigger function processed message: {messageContents}" );
         }
      }
   }
}
```

In the code above, the Application Insights Serilog sink is no longer reinitialized on each function invocation. The `Lazy` field handles initializing the logger if one isn't already available. With this refactoring, the function consistently uses only ~40 threads. Additionally, the thread count is not impacted by additional load.


### Takeaways
To help your azure function scale and to avoid consuming extra resources, reuse as much as possible between invocations.
  * Use static clients to optimize connections. Examples of clients that should be reused are Http Client, Azure Storage clients, and logging clients.
  * Avoid reinitializing classes that can be reused. In my case, it was logging with Serilog but configuration is another candidate to be initialized once for all invocations of a function.
