---
layout: post
title: Effective logging in .Net
description: ''
category: null
tags: []
---

{% include JB/setup %}

Logging is one of those things that everyone says you should be doing but it's hard to get it right. Good logging can be the difference between finding and fixing a bug in a few hours or that same fix taking days and additional releases to isolate a problem. In this post I'll lay out my thoughts on logging and how to do it well. Hat tip to [Nicholas Blumhardt](https://nblumhardt.com/) for introducing many of these ideas to me.

#### Where are we?
The first question to ask is "what kind thing am I building?". If the answer is a .net application or website then you can skip this section. If you're building a reusable library then you should consider an abstraction to let the consumer of your library decide how to log. The .net ecosystem has many options for logging abstraction but most require you to include a reference to the library in your dependency tree. For a cross cutting concern like logging you may run into libraries that use different logging abstractions or potentially conflicting versions of the same abstraction. For this reason I don't like solutions that require a dependency on an external library. I also would prefer to not reinvent the wheel and create a new logging abstraction in each project I work on.

Enter [LibLog](https://github.com/damianh/LibLog), "a single file for you to either copy/paste or install via nuget, into your library/framework/application to provide a logging abstraction." LibLog is just a source file that is included in your library that adds an `internal` `ILog` abstraction. Consumers of your library do not have to care about it because it also includes transparent support for common logging providers. Using reflection LibLog finds the logging framework your application is using and logs to it, without the consumer having to write any code.

One last piece of advice when adding logging a library, make sure you keep the log levels low. It's unlikely you need to log above a `Debug` level. Instead of logging with `Warning` or `Error` levels, communicate the problem in an obvious way using an exception or error code. Even `Information` logs should be kept to a minimum.

#### Application Logging
The more common scenario for developers is the need to log inside of an application like a desktop app or website. For that I prefer structured logging as opposed to text logging. Structured logs rather than simply being a string of text are a set of key-value properties with a timestamp. Where you might have a text log of `"12-17-2016 - Logging in as user with id '1234' failed."`, a structured log would look like `'{ timestamp: '12-17-2016', UserId: 1234, action: 'log_in' status: 'failed'}'`. The structured log conveys the same information (and can even be rendered as a string) but has the advantage of being queryable if stored in a log server that supports structured logs. Rather than searching for a string, you can quickly find all `log_in` actions with a status of `failed` using a query. [Application Insights](https://azure.microsoft.com/en-us/services/application-insights/) and [Seq](https://getseq.net) are two examples of logging servers that support structured logs.

[Serilog](https://serilog.net) is my preferred library for writing structured logs. Serilog pioneered an easy way to write structured logs in .net. Rather than providing an object, you provide a message template similar to a `string.Format` template that looks something like `Log.Information("{Action} finished with {Status} for {UserId}", Actions.LogIn, Statuses.Failed, userId);`. Serilog can then convert that log into a string like `"action 'log_in' finished with status 'failed' for user id '1234'"` or if your logging server supports it, a log object like  

~~~ json
{
    "time": "2016-12-17",
    "level": "Information",
    "template": "{Action} finished with {Status} for {UserId}",
    "properties": {
        "action": "log_in",
        "status": "failed",
        "userId": "1234"
    }
}
~~~

Structured logging using message templates are also supported directly in .net core using the `Microsoft.Extensions.Logging` library. More information on structured logging can be found in [Nicholas Blumhardt's blog series](https://nblumhardt.com/2016/06/structured-logging-concepts-in-net-series-1/).

#### Correlation
When trying to track down an issue with a production application, it's hard to tell which logs go with a given request unless you tie them together. This is commonly called a `CorrelationId`. Serilog supports adding it to all messages via `LogContext` -

~~~csharp
using (LogContext.PushProperty("CorrelationId", Request.Id))
{
  // All logs inside the context will have the CorrelationId added to them
}
~~~
`Microsoft.Extensions.Logging` also supports adding properties to all messages through `ILogger.BeginScope()`. ASP.Net core will even add this by default. You can even use a library like [CorrelationId](https://github.com/stevejgordon/CorrelationId) to ensure a CorrelationId is passed among services if you are running in a microservice environment where you want to track a request across services. In addition to tracking CorrelationId, things like Environment and MachineName can be helpful when troubleshooting.

#### Keep it DRY
Rather than sprinkling `Log.Information(...)` calls throughout your codebase I suggest creating a simple, generic class to encapsulate the logger (Credit to [Erik Heemskerk](https://www.erikheemskerk.nl/meaningful-logging-and-metrics/) for the idea) -

~~~csharp
public class ApplicationMonitoring
{
  // from serilog
  ILogger Logger { get; }
  ILogContext LogContext { get; }
}
~~~
This class is deliberately kept simple so that it can be shared widely throughout your application's code base. Then extend the class with whatever logging needs you might have using extension methods on the `ApplicationMonitoring` class -

~~~csharp
public static class UserActionsApplicationMonitoring
{
   public static void UserLogInFailed( this ApplicationMonitoring monitoring, int userId )
   {
       monitoring.Logger.Information("{Action} finished with {Status} for {UserId}", Actions.LogIn, Statuses.Failed, userId);
   }

   public static void UserLogInSucceeded( this ApplicationMonitoring monitoring, int userId )
   {
       monitoring.Logger.Debug("{Action} finished with {Status} for {UserId}", Actions.LogIn, Statuses.Success, userId);
   }
}
~~~

This way the details of how you log are separate from your business logic. Additionally, if you want to add metrics tracking of your monitoring events it's easy to add to the existing classes. Add an `IMetrics` property to your `ApplicationMonitoring` class then call it from the existing methods. For example, to track the count of failed logins change the `UserLogInFailed` method to -

~~~csharp
   public static void UserLogInFailed( this ApplicationMonitoring monitoring, int userId )
   {
       monitoring.Logger.Information("{Action} finished with {Status} for {UserId}", Actions.LogIn, Statuses.Failed, userId);
       monitoring.Metrics.IncrementCounter( Counters.LogInFailed );
   }
~~~

The last thing that's necessary for effective logs is a way to change logging levels on the fly to tap into extra detail for troubleshooting a production issue. How to do so is outside the scope of this post but it should be easy to do and should not require a restart of the application.

I hope this post gave some clarity on effective logging in .net. Logging is an incredibly effective tool for troubleshooting a production application but it takes work. Instead of leaving it to the end of your project, I suggest laying the groundwork for logs in the beginning to set yourself up for success. Make it easy to write good logs to help developers on your project "fall into the pit of success".
