---
layout: "post"
title: "Stream processing windowing using Azure Event Hubs"
description: ''
category: null
tags: []
---

This post is aimed at engineers designing systems that need to process streams of events. One particular solution is explored, using an Azure Event Hub and the [Capture feature](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-capture-overview).  

There are a few concepts that are helpful to understand before diving in.

1. *What is stream processing?*  
    Stream processing as defined by [Martin Kleppman in his book Designing Data-Intensive Applications](https://dataintensive.net) is
    > somewhere between online and offline/batch processing (so it is sometimes called near-real-time or nearline processing). Like a batch processing system, a stream processor consumes inputs and produces outputs (rather than responding to requests). However, a stream job operates on events shortly after they happen, whereas a batch job operates on a fixed set of input data. This difference allows stream processing systems to have lower latency than the equivalent batch systems.

1. *What is stream windowing?*  
    Windowing, as used in this post, is the process of breaking down events from a continuous stream into groups that occurred during a given time interval (typically small, on the order of minutes or seconds). One iteration of the time interval is considered a "window".

1. *What are Azure Event Hubs?*  
    [Azure Event Hub](https://azure.microsoft.com/en-us/services/event-hubs/) is a managed service that provides a log-based message broker to send and receive messages. A log-based message broker like Event Hub maintains an append only storage of messages and consumers of the log maintain their position in the log as they process messages. Messages are only removed from the log when the log is compacted to remove duplicate messages or when the log begins to run out of space. [Apache Kafka](https://kafka.apache.org) is another common implementation of a log-based message broker with similar semantics.
    
    This log message broker should not be confused with "standard" message brokers like RabbitMQ or Azure Service Bus which provide queueing semantics. Messages are pulled from a queue, then locked by the broker until the message is processed/deleted or returned to the queue. No central log is maintained.


#### Getting started with Capture

A common approach to dealing with an "infinite" event stream is to break it down into time-based windows and process each window as a batch. Any message broker can handle this concept but typically you have to implement it yourself. I like to offload logic like that to the platform, which we can do if we're using Event Hubs. The Event Hub service provides a feature called Capture that will process messages in a configurable window and write them in a batch to either Azure Data Lake Store or Azure Blob Storage.

For the rest of this post I'm assuming that your Event Hub [has been configured to output to Blob Storage using a 5 minute/300 megabyte window](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-capture-enable-through-portal). When Capture is configured, all messages inside a given Event Hub partition and time window will be written to one blob file compressed using the [Avro format](http://avro.apache.org) for efficiency. The naming convention for the blob is __{AzureNamespaceName}/{EventHubName}/{PartitionId}/{DateDownToTheSecond}.avro__. If no messages are received in a window, the Capture feature will write an empty Avro file to blob by default. I recommend turning this off to make processing of the blobs easier.

#### Processing the windowed output

To handle the Capture output using a scalable platform is essential. For big data analysis, Azure Data Lake works well. However, it can be overkill for smaller datasets and lower throughput services. An option for the low-scale case is Azure Functions with a blob trigger like the one below:

```csharp
[FunctionName("CaptureBlobTrigger")]        
public static async Task RunAsync([BlobTrigger("myblobcontainername/{name}.avro")] Stream capturedWindowStream, string name, ILogger log)
{
    await ProcessWindowAsync( capturedWindowStream, log );
}
```

The function will run every time a new blob appears in the blob container `myblobcontainername`. The `ProcessWindowAsync` method can then deserialize the input blob and process it. Microsoft provides the Microsoft.Avro.Core nuget package for working with Avro files as shown below. The code deserializes the input and logs each unique message body contained in the window:

```csharp
private async Task ProcessWindowAsync( Stream capturedWindowStream, ILogger log )
{
  // based on https://gist.github.com/pshrosbree/74c8c4b4744c00cf3d92939952808d1e
  using ( IAvroReader<object> reader = AvroContainer.CreateGenericReader( stream ) )
  {
    while ( reader.MoveNext() )
    {
      foreach ( string recordBody in reader.Current
                    .Objects.Select( o => new AvroEventData( o ) )
                    // assumes UTF8 encoding of input message string
                    .GroupBy( r => Encoding.UTF8.GetString( r.Body ) )
                    .Select( g => g.Key ) )
      {
          log.WriteInformation( $"{DateTime.Now} > Read Unique Item: {recordBody}" );
      }
    }
  }
}

private struct AvroEventData
{
  public AvroEventData( dynamic record )
  {
    SequenceNumber = (long) record.SequenceNumber;
    Offset = (string) record.Offset;
    DateTime.TryParse( (string) record.EnqueuedTimeUtc, out var enqueuedTimeUtc );
    EnqueuedTimeUtc = enqueuedTimeUtc;
    SystemProperties = (Dictionary<string, object>) record.SystemProperties;
    Properties = (Dictionary<string, object>) record.Properties;
    Body = (byte[]) record.Body;
  }
  public long SequenceNumber { get; set; }
  public string Offset { get; set; }
  public DateTime EnqueuedTimeUtc { get; set; }
  public Dictionary<string, object> SystemProperties { get; set; }
  public Dictionary<string, object> Properties { get; set; }
  public byte[] Body { get; set; }
}
```

For a production system, instead of logging, the messages in the window could be processed and analyzed. I hope this post helped you learn about Event Hub Capture and how it might be useful for processing event streams.