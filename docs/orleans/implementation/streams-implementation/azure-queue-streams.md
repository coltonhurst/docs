---
title: Azure Queue streams overview
description: Explore the streaming implementation with Azure Queue in .NET Orleans.
ms.date: 05/23/2025
ms.topic: conceptual
---

# Azure Queue streams overview

Each stream provider (Azure Queues, EventHub, SMS, SQS, etc.) has its own queue-specific details and configuration. This section provides details about the usage, configuration, and implementation of **Orleans Azure Queue Streams**. This section isn't comprehensive. You can find more details in the streaming tests, which contain most configuration options, specifically [`AQClientStreamTests`](https://github.com/dotnet/orleans/tree/main/test/Extensions/TesterAzureUtils/Streaming/AQClientStreamTests.cs) and [`AQSubscriptionMultiplicityTests`](https://github.com/dotnet/orleans/tree/main/test/Extensions/TesterAzureUtils/Streaming/AQSubscriptionMultiplicityTests.cs), and the extension functions for <xref:Orleans.Hosting.IAzureQueueStreamConfigurator> and <xref:Orleans.Hosting.ISiloPersistentStreamConfigurator>.

Orleans Azure Queue requires the [Microsoft.Orleans.Streaming.AzureStorage](https://www.nuget.org/packages/Microsoft.Orleans.Streaming.AzureStorage) NuGet package. In addition to the implementation, the package contains extension methods that simplify configuration at silo startup. The minimal configuration requires specifying the connection string, for example:

```csharp
hostBuilder
    .AddAzureQueueStreams("AzureQueueProvider", configurator =>
    {
        configurator.ConfigureAzureQueue(
            ob => ob.Configure(options =>
            {
                options.ConnectionString = "[PLACEHOLDER]";
                options.QueueNames = new List<string> { "yourprefix-azurequeueprovider-0" };
            }));
    configurator.ConfigureCacheSize(1024);
    configurator.ConfigurePullingAgent(ob => ob.Configure(options =>
    {
      options.GetQueueMsgsTimerPeriod = TimeSpan.FromMilliseconds(200);
    }));
  })
  // a PubSubStore could be needed, as example Azure Table Storage
  .AddAzureTableGrainStorage("PubSubStore", options => {
    options.ConnectionString = "[PLACEHOLDER]";
  })
```

The pulling agents pull repeatedly until there are no more messages on a queue, then delay for a configurable period before continuing to pull. This process occurs for each queue. Internally, the pulling agents place messages in a cache (one cache per queue) for delivery to consumers but stop reading if the cache fills up. Messages are removed from the cache once consumers process them, so the read rate should be roughly throttled by the consumers' processing rate.

By default, Orleans Azure Queue uses **8 queues** (see <xref:Orleans.Configuration.AzureQueueOptions>) and 8 related pulling agents, a delay of **100 ms** (see <xref:Orleans.Configuration.StreamPullingAgentOptions.GetQueueMsgsTimerPeriod?displayProperty=nameWithType>), and a cache size (<xref:Orleans.Streams.IQueueCache>) of **4096 messages** (see <xref:Orleans.Configuration.SimpleQueueCacheOptions.CacheSize?displayProperty=nameWithType>).

## Configuration

The default configuration should fit a production environment, but for special needs, it's possible to configure the default behavior. For example, in a development machine, it's possible to reduce the number of polling agents to using just one queue. This can help to reduce CPU usage and resource pressure.

```csharp
hostBuilder
    .AddAzureQueueStreams<AzureQueueDataAdapterV2>(
        "AzureQueueProvider",
        optionsBuilder => optionsBuilder.Configure(
            options =>
            {
                options.ConnectionString = "[PLACEHOLDER]";
                options.QueueNames =
                    new List<string>
                    {
                        "yourprefix-azurequeueprovider-0"
                    };
            }))
```

## Tuning

In a production system, you may need to tune the default configuration. When tuning some factors should be considered, and it's service-specific.

1. First, most of the settings are per queue, so for a large number of streams, the load on each queue can be reduced by configuring more queues.
1. Since streams process messages in order per stream, the gating factor will be the number of events being sent on a single stream.
1. A reasonable balance of **cache time** (<xref:Orleans.Configuration.StreamPullingAgentOptions.GetQueueMsgsTimerPeriod?displayProperty=nameWithType>) versus **visibility time** (<xref:Orleans.Configuration.AzureQueueOptions.MessageVisibilityTimeout?displayProperty=nameWithType>) is that the visibility should be configured **to double** the time messages **are expected** to be in the cache.

### Example

Assuming a system with these characteristics:

- 100 streams,
- 10 queues,
- Each stream processing 60 messages per minute,
- Each message takes around 30ms to process,
- 1 minute worth of messages in cache (cache time).

So we can calculate some parameters of the system:

- Streams/queue: Even balancing of streams across queues would be an ideal 10 streams/queue (100 streams / 10 queues).
But since streams won't always be evenly balanced over the queues, doubling (or even tripling) the ideal is safer than expecting ideal distribution.
Hence **20 streams/queue** (10 streams/queue x 2 as safety factor) is probably reasonable.

- Messages/minute: This means each queue will be expected to process up to **1200 messages/minute** (60 messages x 20 streams).

Then we can determine the visibility time to use:

- Visibility time: The cache time (1 minute) is configured to hold 1 minute of messages (so 1200 messages, as we calculated messages/minute above).
We assumed that each message takes 30 ms to process, then we can expect messages to spend up to 36 seconds in the cache (0.030 sec x 1200 msg = 36 sec), so the visibility time - doubled for safety - would need be over 72 seconds (36 sec of time in cache x 2).
Accordingly, if we define a bigger cache, that would require a longer visibility time.

Final considerations in a real-world system:

- Since the order is only per stream, and a queue consumes many streams, messages will likely be processed across multiple streams in parallel (as an example: we have a grain for the stream, which can run in parallel). This means we'll burn through the cache in far less time, but we planned for the worse case: it will give the system room to continue to function well even under intermittent delays and transient errors.

So we can configure Azure Queue Streams using:

```csharp
hostBuilder
  .AddAzureQueueStreams("AzureQueueProvider", configurator => {
    configurator.ConfigureAzureQueue(
      ob => ob.Configure(options => {
        options.ConnectionString = "[PLACEHOLDER]";
        options.QueueNames = new List<string> {
          "yourprefix-azurequeueprovider-1",
          [...]
          "yourprefix-azurequeueprovider-10",
        };
        options.MessageVisibilityTimeout = TimeSpan.FromSeconds(72);
      }));
    configurator.ConfigureCacheSize(1200);
  })
```
