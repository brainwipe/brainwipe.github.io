---
layout: post
title: "Brighter Servicehost as a .NET Core 2.x linux daemon with RabbitMQ"
date: 2018-07-11 0:00:00 +01:00
categories: brighter, cqrs, dotnet, core, linux
---
We're going to implement a simple linux daemon service in .NET Core 2.x that hosts a Brighter Service for responding to events posted to a RabbitMQ queue.

The [Brighter command processor library](https://www.goparamore.io) for .NET allows you to decouple the endpoint from the domain. When using the library with a message broker (such as [RabbitMQ](https://www.rabbitmq.com/)), you can use a separate service to watch the message queue and respond to their messages. The Brighter [Greetings Example](https://www.goparamore.io/greetings-example/) shows you how to host a Brighter Service Activator in a windows server (using a console app and TopShelf). 

## Install RabbitMQ
Surf to the [RabbitMQ](https://www.rabbitmq.com/) website, download the latest Erlang (programming environment that Rabbit runs on) and installer. Take the default options for each. 

Once installed:
- In powershell, enable the management plugin: `rabbitmq-plugins enable rabbitmq_management`
- Go to `http://localhost:15672`
- Username `guest` and password `guest`
- Click `Exchanges` tab and add a new exchange with the following parameters:
  - Name: `my.exchange`
  - Type: `fanout`
  - Durability: `Durable`
  - Auto delete: No
  - Internal: No
  - Leave arguments blank
  - Click Add Exchange
- Click `Queues` tab at the top, click Add Queue with the following parameters
  - Name: `my.queue`
  - Durability: `Durable`
  - Arguments: leave blank
  - Click Add Queue

We now how to connect the exchange to the queue. In the list of Queues:
 - Click on your queue called `my.queue`
 - Expand the `bindings` sub section.
 - From exchange: `my.exchange`
 - Leave all other boxes blank and click Bind.

 You can now post events in the exchange window and they will appear in the queue.

## Create a .NET Core Linux Daemon
We're going to create a .NET Core 2.x console application that can be run as a service on linux (a daemon). It can also be run via Powershell/Visual Studio for testing.

- Ensure you have the [.NET Core 2.2](https://dotnet.microsoft.com/download/dotnet-core/2.2) (or later) SDK installed.
- Open VS2017, create a new .NET Core Console Project
- Once created, open the project properties, select Build, go to Advanced and select `C# latest minor version`. We need this for the `static async Main` below.
- Using nuget, add the following packages (and their dependencies):
  - `Microsoft.Extensions.Configuration.CommandLine`
  - `Microsoft.Extensions.Configuration.EnvironmentVariables`
  - `Microsoft.Extensions.DependencyInjection`
  - `Microsoft.Extensions.Hosting`
  - `Microsoft.Extensions.Logging.Console`
  - `Microsoft.Extensions.Options.ConfigurationExtensions`
- Open Program, remove the Main method and replace it with this Daemon boilerplate:

```csharp
    public static async Task Main(string[] args)
    {
        var builder = new HostBuilder()
            .ConfigureAppConfiguration((hostingContext, config) =>
            {
                config.AddEnvironmentVariables();

                if (args != null)
                {
                    config.AddCommandLine(args);
                }
            })
            .ConfigureServices((hostContext, services) =>
            {
                services.AddOptions();
                services.Configure<DaemonConfig>(hostContext.Configuration.GetSection("Daemon"));
                services.AddSingleton<IHostedService, DaemonService>();
            })
            .ConfigureLogging((hostingContext, logging) => {
                logging.AddConfiguration(hostingContext.Configuration.GetSection("Logging"));
                logging.AddConsole();
            });

        await builder.RunConsoleAsync();

    }
```

Don't worry about `Daemon Config` or `DaemonService`, we're about to create those. All the other types and methods should be included in using statements.

### Configuration
We want to pass configuration into the daemon as commandline parameters. In the `Main` method, `ConfigureAppConfiguration` binds any commandline arguments to a configuration section called "Daemon". We need to create a class to hold the command line arguments, we're going to call it `DaemonConfig`. In the future, if you need to add more command line arguments, then all you need to do is add properties to this class.

```csharp
    public class DaemonConfig
    {
        public string DaemonName { get; set; }
    }
```

### Service
We're going to add a temporary service so that you can see the daemon running in its simplest form. We'll replace this with the Brigher one later. The service implements `IHostedService` and accepts events from host operating system for Starting, Stopping and so on. The configuration we've just created is injected in the constructor for us to use.

```csharp
    public class DaemonService : IHostedService, IDisposable
    {
        private readonly ILogger logger;
        private readonly IOptions<DaemonConfig> config;

        public DaemonService(ILogger<DaemonService> logger, IOptions<DaemonConfig> config)
        {
            this.logger = logger;
            this.config = config;
        }

        public Task StartAsync(CancellationToken cancellationToken)
        {
            logger.LogInformation("Starting: " + config.Value.DaemonName);
            return Task.CompletedTask;
        }

        public Task StopAsync(CancellationToken cancellationToken)
        {
            logger.LogInformation("Stopping: " + config.Value.DaemonName);
            return Task.CompletedTask;
        }

        public void Dispose()
        {
            logger.LogInformation("Disposing " + config.Value.DaemonName);
        }
    }
```

## First Run
Build the code you have so far, open powershell and cd into the project folder, run the daemon using:

`dotnet run --Daemon:DaemonName="Hello World!"`

Press CTRL-C to stop the daemon. My console is called MCO.Reporting.Console, so I see:

```powershell
[11:33:54] MCO.Reporting.Console> dotnet run --Daemon:DaemonName="Hello World!"
Application started. Press Ctrl+C to shut down.
Hosting environment: Production
infoContent root path: C:\projects\MCO-Reporting\src\MCO.Reporting.Console\bin\Debug\netcoreapp2.2\
: MCO.Reporting.Console.DaemonService[0]
      Starting: Hello World!
info: MCO.Reporting.Console.DaemonService[0]
      Stopping: Hello World!
info: MCO.Reporting.Console.DaemonService[0]
      Disposing Hello World!
[11:33:59] MCO.Reporting.Console>
```

## Adding Brighter
We're now going to add Brighter to watch for messages on our Rabbit MQ queue and print them to the console.

- Select your project, manage nuget packages and add the following packages (and their dependencies):
  - `Paramore.Brighter.ServiceActivator`
  - `Paramore.Brighter.ServiceActivator.Extensions.DependencyInjection`
  - `Paramore.Brighter.ServiceActivator.Extensions.Hosting`
  - `Paramore.Brighter.MessagingGateway.RMQ`
  - `Paramore.Brighter.Extensions.DependencyInjection`
- 

- NB: you can customise Brighter by passing an anonymous type when calling `AddBrighter()` [see the docs!](https://github.com/BrighterCommand/Paramore.Brighter.Extensions#1-paramorebrighterextensionsdependencyinjection)

