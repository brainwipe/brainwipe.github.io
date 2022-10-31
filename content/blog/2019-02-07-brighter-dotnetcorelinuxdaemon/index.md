---
title: "Brighter Servicehost as a .NET Core 2.x linux daemon with RabbitMQ"
date: 2018-07-11
categories: brighter cqrs dotnet core linux
---
We're going to implement a simple linux daemon service in .NET Core 2.x that hosts a Brighter Service for responding to events posted to a RabbitMQ queue. If you just want to create a Linux daemon, see my [post on creating a simple dotnet Core linux daemon](/2019-02-05-dotnetcore-linux-daemon).

The [Brighter command processor library](https://www.goparamore.io) for .NET allows you to decouple the endpoint from the domain. When using the library with a message broker (such as [RabbitMQ](https://www.rabbitmq.com/)), you can use a separate service to watch the message queue and respond to their messages. The Brighter [Greetings Example](https://www.goparamore.io/greetings-example/) shows you how to host a Brighter Service Activator in a windows server (using a console app and TopShelf). 

We're not going to cover how to create commands and events and posting them to the queue, you can find details of that in the [Sender of the Paramore Extensions Greetings example](https://github.com/BrighterCommand/Paramore.Brighter.Extensions/blob/master/samples/GreetingsSender/Program.cs).

## Install RabbitMQ
Surf to the [RabbitMQ](https://www.rabbitmq.com/) website, download the latest Erlang (programming environment that Rabbit runs on) and installer. Take the default options for each. 

Once installed:
- In powershell, enable the management plugin: `rabbitmq-plugins enable rabbitmq_management`
- Go to `http://localhost:15672`
- Username `guest` and password `guest`

We do not need to create exchanges, queues or bindings in the management UI, Brighter will perform that task for us.

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
            })
            .ConfigureLogging((hostingContext, logging) => {
                logging.AddConfiguration(hostingContext.Configuration.GetSection("Logging"));
                logging.AddConsole();
            });

        await builder.RunConsoleAsync();

    }
```

There is no service or configuration yet, we're going to let Brighter's [extensions](https://github.com/BrighterCommand/Paramore.Brighter.Extensions) set those up for us.

## Adding Brighter
We're now going to add Brighter to watch for messages on our Rabbit MQ queue and print them to the console.

- Select your project, manage nuget packages and add the following packages (and their dependencies):
  - `Paramore.Brighter.ServiceActivator`
  - `Paramore.Brighter.ServiceActivator.Extensions.DependencyInjection`
  - `Paramore.Brighter.ServiceActivator.Extensions.Hosting`
  - `Paramore.Brighter.MessagingGateway.RMQ`
  - `Paramore.Brighter.Extensions.DependencyInjection`

## Creating the Rabbit Gateway Connection
At the top of the `Main` we're going to create the connection to Rabbit. `Name` allows us to identify what is connected in the management UI, the AMPQ Uri is the connection string with `guest` for username and password (the default). The Exchange is the "port" which acts as an interface between the queues and our consumer console app.

```csharp
    var rmqConnection = new RmqMessagingGatewayConnection
    {
        Name = "My Linux Daemon",
        AmpqUri = new AmqpUriSpecification(new Uri("amqp://guest:guest@localhost:5672")),
        Exchange = new Exchange("my.exchange", "fanout", true),
    };
    var rmqMessageConsumerFactory = new RmqMessageConsumerFactory(rmqConnection);
```

After that, we need to build a connection for each of the events that we are going to consume. The connection has an optional `ConnectionName`. The `ChannelName` will become the name of the queue in RabbitMQ. Durable queues are those that remember your messages (events and commands) even if RabbitMQ (sometimes called The Node) is restarted.l

```csharp
    var connection = new Connection<MyEvent>(
        new ConnectionName("MCO.Reporting.Console"),
        new ChannelName("my.qeue"),
        timeoutInMilliseconds: 200,
        isDurable: true,
        highAvailability: true);
```

- NB: you can customise Brighter by passing an anonymous type when calling `AddBrighter()` [see the docs!](https://github.com/BrighterCommand/Paramore.Brighter.Extensions#1-paramorebrighterextensionsdependencyinjection)

