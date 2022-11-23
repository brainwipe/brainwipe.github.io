---
title: Troubleshooting Connecting .NET to MSK
date: 2022-11-02
categories: dotnet kafka aws msk openssl
---

In my [previous blog post](../2022-10-31-aws-msk-dotnet), I looked at connecting a .NET client to a Kafka cluster on MSK.

To get to that point, I hit a lot of errors that weren't immediately obvious what the cause was. Here's how I troubleshooted them, I hope this is useful to you! 

Most of the issues I had was around ensuring that MSK was correctly configured to be both secure and... well, accessible!

## Assumptions

Before we start, I assume that you have a .NET client running in a command line like the one in the [last blog post](../2022-10-31-aws-msk-dotnet) running but it doesn't seem to connect to MSK.

## Turn on debugging first!

A typical .NET client configuration looks like this:

```csharp
var adminClientConfig = new AdminClientConfig
{
    BootstrapServers = bootstrap,
    SecurityProtocol = SecurityProtocol.Ssl,
    SslCertificateLocation = "kafka_test_client.cer",
    SslKeyLocation = "kafka_test_client.key"
};
```

Very little information will written out to the console, so you need to turn on debugging:

```csharp
var adminClientConfig = new AdminClientConfig
{
    Debug = "broker,protocol,metadata,topic", // Add Debugging Line
    BootstrapServers = bootstrap,
    SecurityProtocol = SecurityProtocol.Ssl,
    SslCertificateLocation = "kafka_test_client.cer",
    SslKeyLocation = "kafka_test_client.key"
};
```

When running the client, you will see a huge amount of information. You might want to reduce or switch off for production.

## Error: Terminating instance (destroy flags none (0x0))

Below is a debug output for mutual TLS. The output is from a C/C++ library called [`librdkafka`](https://www.nuget.org/packages/librdkafka.redist), which the .NET client binds to.

There are two brokers (`broker1-msk.amazonaws.com:9194/bootstrap` and `broker2-msk.amazonaws.com:9194/bootstrap`) - your URLs will look a little different, I've adjusted mine to protect the innocent!

The client is trying to connect via SSL to each broker (`TRY_CONNECT -> CONNECT`). It keeps trying with `Cluster connection already in progress: no cluster connection` and eventually you see `Terminating instance (destroy flags none (0x0))`. The rest of the trace is client cleaning up after itself.

This means that the client cannot *reach* the MSK brokers.

```log
%7|1666628222.105|BROKER|rdkafka#producer-1| [thrd:app]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Added new broker with NodeId -1
%7|1666628222.105|BROKER|rdkafka#producer-1| [thrd:app]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Added new broker with NodeId -1
%7|1666628222.105|BRKMAIN|rdkafka#producer-1| [thrd::0/internal]: :0/internal: Enter main broker thread
%7|1666628222.105|CONNECT|rdkafka#producer-1| [thrd:app]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Selected for cluster connection: bootstrap servers added (broker has 0 connection attempt(s))
%7|1666628222.105|BRKMAIN|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Enter main broker thread
%7|1666628222.106|CONNECT|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Received CONNECT op
%7|1666628222.106|STATE|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Broker changed state INIT -> TRY_CONNECT
%7|1666628222.106|CONNECT|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: broker in state TRY_CONNECT connecting
%7|1666628222.106|STATE|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Broker changed state TRY_CONNECT -> CONNECT
%7|1666628222.105|BRKMAIN|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Enter main broker thread
%7|1666628222.106|INIT|rdkafka#producer-1| [thrd:app]: librdkafka v1.9.2 (0x10902ff) rdkafka#producer-1 initialized (builtin.features gzip,snappy,ssl,sasl,regex,lz4,sasl_gssapi,sasl_plain,sasl_scram,plugins,zstd,sasl_oauthbearer,http,oidc, SSL ZLIB SNAPPY ZSTD CURL SASL_SCRAM SASL_OAUTHBEARER PLUGINS HDRHISTOGRAM, debug 0x8e)
%7|1666628222.117|CONNECT|rdkafka#producer-1| [thrd:app]: Not selecting any broker for cluster connection: still suppressed for 37ms: application metadata request
%7|1666628222.146|CONNECT|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Connecting to ipv4#xxx.xxx.xxx.xxx:9194 (ssl) with socket 2744
%7|1666628223.145|CONNECT|rdkafka#producer-1| [thrd:main]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Selected for cluster connection: no cluster connection (broker has 0 connection attempt(s))
%7|1666628223.145|CONNECT|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Received CONNECT op
%7|1666628223.147|STATE|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Broker changed state INIT -> TRY_CONNECT
%7|1666628223.148|CONNECT|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: broker in state TRY_CONNECT connecting
%7|1666628223.148|STATE|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Broker changed state TRY_CONNECT -> CONNECT
%7|1666628223.148|CONNECT|rdkafka#producer-1| [thrd:app]: Not selecting any broker for cluster connection: still suppressed for 46ms: application metadata request
%7|1666628223.149|CONNECT|rdkafka#producer-1| [thrd:app]: Not selecting any broker for cluster connection: still suppressed for 45ms: application metadata request
%7|1666628223.183|CONNECT|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Connecting to ipv4#xxx.xxx.xxx.xxx:9194 (ssl) with socket 2816
%7|1666628224.178|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628225.222|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628226.284|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628227.329|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628228.381|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628229.419|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628230.454|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628231.501|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628232.461|CONNECT|rdkafka#producer-1| [thrd:app]: Cluster connection already in progress: application metadata request
%7|1666628232.538|CONNECT|rdkafka#producer-1| [thrd:main]: Cluster connection already in progress: no cluster connection
%7|1666628232.805|DESTROY|rdkafka#producer-1| [thrd:app]: Terminating instance (destroy flags none (0x0))
%7|1666628232.806|DESTROY|rdkafka#producer-1| [thrd:main]: Destroy internal
%7|1666628232.806|DESTROY|rdkafka#producer-1| [thrd:main]: Removing all topics
%7|1666628232.806|DESTROY|rdkafka#producer-1| [thrd:main]: Sending TERMINATE to ssl://broker2-msk.amazonaws.com:9194/bootstrap
%7|1666628232.807|DESTROY|rdkafka#producer-1| [thrd:main]: Sending TERMINATE to ssl://broker1-msk.amazonaws.com:9194/bootstrap
%7|1666628232.807|TERM|rdkafka#producer-1| [thrd::0/internal]: :0/internal: Received TERMINATE op in state INIT: 2 refcnts, 0 toppar(s), 0 active toppar(s), 0 outbufs, 0 waitresps, 0 retrybufs
%7|1666628232.807|FAIL|rdkafka#producer-1| [thrd::0/internal]: :0/internal: Client is terminating (after 10702ms in state INIT) (_DESTROY)
%7|1666628232.807|STATE|rdkafka#producer-1| [thrd::0/internal]: :0/internal: Broker changed state INIT -> DOWN
%7|1666628232.807|BRKTERM|rdkafka#producer-1| [thrd::0/internal]: :0/internal: terminating: broker still has 2 refcnt(s), 0 buffer(s), 0 partition(s)
%7|1666628232.807|TERM|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Received TERMINATE op in state CONNECT: 2 refcnts, 0 toppar(s), 0 active toppar(s), 0 outbufs, 0 waitresps, 0 retrybufs
%7|1666628232.807|TERM|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Received TERMINATE op in state CONNECT: 2 refcnts, 0 toppar(s), 0 active toppar(s), 0 outbufs, 0 waitresps, 0 retrybufs
%7|1666628232.807|TERMINATE|rdkafka#producer-1| [thrd::0/internal]: :0/internal: Handle is terminating in state DOWN: 1 refcnts (0000022DACAE1508), 0 toppar(s), 0 active toppar(s), 0 outbufs, 0 waitresps, 0 retrybufs: failed 0 request(s) in retry+outbuf
%7|1666628232.807|FAIL|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Client is terminating (after 10701ms in state CONNECT) (_DESTROY)
%7|1666628232.807|STATE|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Broker changed state CONNECT -> DOWN
%7|1666628232.807|FAIL|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Client is terminating (after 9658ms in state CONNECT) (_DESTROY)
%7|1666628232.807|STATE|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Broker changed state CONNECT -> DOWN
%7|1666628232.807|BRKTERM|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: terminating: broker still has 2 refcnt(s), 0 buffer(s), 0 partition(s)
%7|1666628232.807|FAIL|rdkafka#producer-1| [thrd::0/internal]: :0/internal: Broker handle is terminating (after 0ms in state DOWN) (_DESTROY)
%7|1666628232.807|BRKTERM|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: terminating: broker still has 2 refcnt(s), 0 buffer(s), 0 partition(s)
%7|1666628232.808|TERMINATE|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Handle is terminating in state DOWN: 1 refcnts (0000022DACAD8418), 0 toppar(s), 0 active toppar(s), 0 outbufs, 0 waitresps, 0 retrybufs: failed 0 request(s) in retry+outbuf
%7|1666628232.808|TERMINATE|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Handle is terminating in state DOWN: 1 refcnts (0000022DAC23E208), 0 toppar(s), 0 active toppar(s), 0 outbufs, 0 waitresps, 0 retrybufs: failed 0 request(s) in retry+outbuf
%7|1666628232.808|FAIL|rdkafka#producer-1| [thrd:ssl://broker1-msk.amazonaws]: ssl://broker1-msk.amazonaws.com:9194/bootstrap: Broker handle is terminating (after 0ms in state DOWN) (_DESTROY)
%7|1666628232.808|FAIL|rdkafka#producer-1| [thrd:ssl://broker2-msk.amazonaws]: ssl://broker2-msk.amazonaws.com:9194/bootstrap: Broker handle is terminating (after 0ms in state DOWN) (_DESTROY)
```

### Possible Causes

- The port it's using (9194) has not been added to the security group (firewall) for inbound traffic.
- You have not switched on public access. [Here's how](https://docs.aws.amazon.com/msk/latest/developerguide/public-access.html).

## system library:fopen:No such file or directory: fopen

This happens when the library cannot load the certificate that you have specified in your configuration. You get a similar error for an incorrect key file.

For example:

```csharp
var adminClientConfig = new AdminClientConfig
{
    SocketConnectionSetupTimeoutMs = 18000,
    Debug = "broker,protocol,metadata,topic",
    BootstrapServers = bootstrap,
    SecurityProtocol = SecurityProtocol.Ssl,
    SslCertificateLocation = "TYPOkafka_test_client.cer", // TYPO here
    SslKeyLocation = "kafka_test_client.key"
};
```

```log
%3|1667391346.010|SSL|rdkafka#producer-1| [thrd:app]: crypto\bio\bss_file.c:288: error:02001002:system library:fopen:No such file or directory: fopen('TYPOkafka_test.cer','rb')
%3|1667391346.010|SSL|rdkafka#producer-1| [thrd:app]: crypto\bio\bss_file.c:290: error:20074002:BIO routines:file_ctrl:system lib
System.InvalidOperationException: ssl.certificate.location failed: ssl\ssl_rsa.c:596: error:140DC002:SSL routines:use_certificate_chain_file:system lib
  at SafeKafkaHandle Confluent.Kafka.Impl.SafeKafkaHandle.Create(RdKafkaType type, IntPtr config, IClient owner)
  at Confluent.Kafka.Producer`2..ctor(ProducerBuilder<TKey, TValue> builder)
  at Confluent.Kafka.AdminClient..ctor(AdminClientBuilder builder)
  at IAdminClient Confluent.Kafka.AdminClientBuilder.Build()
  at void MCO.CLI.Commands.KafkaCommand.ListTopics() in C:\Projects\MCO\MCO.CLI\Commands\KafkaCommand.cs:61
  at async ValueTask MCO.CLI.Commands.KafkaCommand.ExecuteAsync(IConsole console) in C:\Projects\MCO\MCO.CLI\Commands\KafkaCommand.cs:40
  at async Task Typin.Internal.Pipeline.ExecuteCommand.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken)
  at async Task MCO.CLI.Middleware.CheckZoneMiddleware.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken) in C:\Projects\MCO\MCO.CLI\Middleware\CheckZoneMiddleware.cs:25
  at async Task Typin.Internal.Pipeline.BindInput.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken)
  at async Task Typin.Internal.Pipeline.HandleSpecialOptions.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken)
  at async Task Typin.Internal.Pipeline.ExecuteDirectivesSubpipeline.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken)
  at async Task Typin.Internal.Pipeline.InitializeDirectives.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken)
  at async Task Typin.Internal.Pipeline.ResolveCommandSchemaAndInstance.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken)
  at async Task MCO.CLI.Middleware.TimeTravelMiddleware.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken) in C:\Projects\MCO\MCO.CLI\Middleware\TimeTravelMiddleware.cs:46
  at async Task MCO.CLI.Middleware.CliTenantMiddleware.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken) in C:\Projects\MCO\MCO.CLI\Middleware\CliTenantMiddleware.cs:53
  at async Task MCO.CLI.Middleware.LogLevelMiddleware.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken) in C:\Projects\MCO\MCO.CLI\Middleware\LogLevelMiddleware.cs:25
  at async Task MCO.CLI.Middleware.McoInitialisingMiddleware.HandleAsync(ICliContext context, CommandPipelineHandlerDelegate next, CancellationToken cancellationToken) in C:\Projects\MCO\MCO.CLI\Middleware\McoInitialisingMiddleware.cs:21
  at async Task Typin.Internal.CliCommandExecutor.RunPipelineAsync(IServiceProvider serviceProvider, ICliContext context)
  at async Task<int> Typin.Internal.CliCommandExecutor.ExecuteCommandAsync(IEnumerable<string> commandLineArguments)
  ```

## SSL routines:ssl3_read_bytes:sslv3 alert bad certificate

This happens when your key file was not configured at all.

```csharp
var adminClientConfig = new AdminClientConfig
{
    SocketConnectionSetupTimeoutMs = 18000,
    Debug = "broker,protocol,metadata,topic",
    BootstrapServers = bootstrap,
    SecurityProtocol = SecurityProtocol.Ssl,
    SslCertificateLocation = "kafka_test_client.cer",
    // SslKeyLocation = "kafka_test_client.key" // Key file is not configured!
    };
};
```

```log
%7|1667391595.595|BRKMAIN|rdkafka#producer-1| [thrd::0/internal]: :0/internal: Enter main broker thread
%7|1667391595.595|BROKER|rdkafka#producer-1| [thrd:app]: ssl://broker1.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Added new broker with NodeId -1
%7|1667391595.595|BRKMAIN|rdkafka#producer-1| [thrd:ssl://broker1.kafka.eu-west-2.amazonaws]: ssl://broker1.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Enter main broker thread
%7|1667391595.595|BROKER|rdkafka#producer-1| [thrd:app]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Added new broker with NodeId -1
%7|1667391595.596|CONNECT|rdkafka#producer-1| [thrd:app]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Selected for cluster connection: bootstrap servers added (broker has 0 connection attempt(s))
%7|1667391595.596|BRKMAIN|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Enter main broker thread
%7|1667391595.596|INIT|rdkafka#producer-1| [thrd:app]: librdkafka v1.9.2 (0x10902ff) rdkafka#producer-1 initialized (builtin.features gzip,snappy,ssl,sasl,regex,lz4,sasl_gssapi,sasl_plain,sasl_scram,plugins,zstd,sasl_oauthbearer,http,oidc, SSL ZLIB SNAPPY ZSTD CURL SASL_SCRAM SASL_OAUTHBEARER PLUGINS HDRHISTOGRAM, debug 0x8e)
%7|1667391595.596|CONNECT|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Received CONNECT op
%7|1667391595.596|STATE|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Broker changed state INIT -> TRY_CONNECT
%7|1667391595.596|CONNECT|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: broker in state TRY_CONNECT connecting
%7|1667391595.596|STATE|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Broker changed state TRY_CONNECT -> CONNECT
%7|1667391595.606|CONNECT|rdkafka#producer-1| [thrd:app]: Not selecting any broker for cluster connection: still suppressed for 39ms: application metadata request
%7|1667391595.621|CONNECT|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Connecting to ipv4#3.8.254.176:9194 (ssl) with socket 2548
%7|1667391595.655|CONNECT|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Connected to ipv4#3.8.254.176:9194
%7|1667391595.656|STATE|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Broker changed state CONNECT -> SSL_HANDSHAKE
%7|1667391595.656|CONNECT|rdkafka#producer-1| [thrd:app]: ssl://broker1.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Selected for cluster connection: application metadata request (broker has 0 connection attempt(s))
%7|1667391595.657|CONNECT|rdkafka#producer-1| [thrd:ssl://broker1.kafka.eu-west-2.amazonaws]: ssl://broker1.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Received CONNECT op
%7|1667391595.657|STATE|rdkafka#producer-1| [thrd:ssl://broker1.kafka.eu-west-2.amazonaws]: ssl://broker1.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Broker changed state INIT -> TRY_CONNECT
%7|1667391595.657|CONNECT|rdkafka#producer-1| [thrd:ssl://broker1.kafka.eu-west-2.amazonaws]: ssl://broker1.kafka.eu-west-2.amazonaws.com:9194/bootstrap: broker in state TRY_CONNECT connecting
%7|1667391595.657|STATE|rdkafka#producer-1| [thrd:ssl://broker1.kafka.eu-west-2.amazonaws]: ssl://broker1.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Broker changed state TRY_CONNECT -> CONNECT
%7|1667391595.657|CONNECT|rdkafka#producer-1| [thrd:app]: Not selecting any broker for cluster connection: still suppressed for 49ms: application metadata request
%7|1667391595.657|CONNECT|rdkafka#producer-1| [thrd:app]: Not selecting any broker for cluster connection: still suppressed for 49ms: application metadata request
%7|1667391595.678|CONNECT|rdkafka#producer-1| [thrd:ssl://broker1.kafka.eu-west-2.amazonaws]: ssl://broker1.kafka.eu-west-2.amazonaws.com:9194/bootstrap: Connecting to ipv4#18.169.67.207:9194 (ssl) with socket 2584
%7|1667391595.691|FAIL|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: SSL handshake failed: ssl\record\rec_layer_s3.c:1544: error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate: SSL alert number 42 (after 34ms in state SSL_HANDSHAKE) (_SSL)
%3|1667391595.691|FAIL|rdkafka#producer-1| [thrd:ssl://broker2.kafka.eu-west-2.amazonaws]: ssl://broker2.kafka.eu-west-2.amazonaws.com:9194/bootstrap: SSL handshake failed: ssl\record\rec_layer_s3.c:1544: error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate: SSL alert number 42 (after 34ms in state SSL_HANDSHAKE)

Logs repeat...

```
