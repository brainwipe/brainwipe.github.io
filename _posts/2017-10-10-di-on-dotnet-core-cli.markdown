---
layout: post
title: "Using Dependency Injection on the .NET Core Command Line"
date: 2017-10-10 09:00:00 +01:00
categories: asp.net core cli commandline di
---

Developers should build tools to help them develop code. If you are building an ASP.NET web app then I can recommend building a companion command line tool for you to use. The command line tool is there to help you understand system state or show data in your system in such a way that makes sense for you. It's not for release, it's to make you more effective. It's the one thing I wish I've always done. 

I like to include it in the same solution as the main application so that it can be built and kept up to date with the system. I also use it to call the service layer directly (by-passing the web API) and use the real entity objects. I want to be able to use the same dependency injection registration process in the command line 

I'm going to assume you know what dependency injection is. If not, then please do read a bit of [Martin Fowler](https://martinfowler.com/articles/injection.html) (I'm a bit of a fan).

## Get the code

For those wanting to jump in, check out the [GitHub Repo DIOnDotNetCoreCLI](https://github.com/brainwipe/DIOnDotNetCoreCLI).

## .NET CommandLineUtils

.NET Core gives us a new challenge as we can create command line apps that run on both Linux and Windows, making them better for fitting into your tool chains. Microsoft originally created the `CommandLineUtils` DLL but have discontinued support. Instead Nate McMaster (of the ASP.NET team) [has forked the code](https://github.com/aspnet/Common/pull/261) and will maintain it as a [personal project](https://github.com/natemcmaster/CommandLineUtils). Thank you to Nate for taking on this ace library!

## Anonymous functions vs Separate Command Classes

If you command line application is simple then I recommend using the [anonymous function method](https://github.com/natemcmaster/CommandLineUtils#usage) of building your tools.

I prefer to split out each command into separate classes. Mostly because I tend to have lots of commands in a single console project. For example `c:/>myapp -say "hello world"` on the commandline would be in a class `SayCommand`. 

I'm going to assume that you're following separate command classes for this tutorial.

## 1. Example Console App
For this example, we have a service that prints out a Shakespear Quote (`QuotesService`) and a service that wraps text with some ascii art (`FormatterService`). I want to use a command line application to call an exteronal service library. For most of my commands I want them to be of the form: `c:/>myapp <command name> <options>`. The CommandLineUtils library supports multiple levels but 

- Create a new .NET Core Console app: New Project &rarr; Visual C#/.NET Core/Console App (.NET COre)
- Right click project &rarr; Manage nuget packages
- Click Browse, install `McMaster.Extensions.CommandLineUtils`
- Also install `Microsoft.Extensions.DependencyInjection`

## 2. Add your services
When you're doing this for your application then you add in a reference for your service library. I'm not going to go through how to make one here, you can [check out the code on Github](https://github.com/brainwipe/DIOnDotNetCoreCLI/tree/master/DIOnDotNetCoreCLI.Services). It's the next bit we want to spend time on.

## 3. Create a root Command Line Application
The core class of the CommandLineUtils library is the `CommandLineApplication`. You need to create at least one `CommandLineApplication` and then attach commands to it as children. We're going to create our own version that can handle the dependency injection.

- Create a new class `CommandLineApplicationWithDI`

```csharp
public class CommandLineApplicationWithDI : CommandLineApplication
{
    private readonly IServiceProvider serviceProvider;

    public CommandLineApplicationWithDI(IServiceProvider serviceProvider)
    {
        this.serviceProvider = serviceProvider;
        RegisterCommands();
    }

    private void RegisterCommands()
    {
        foreach (var command in serviceProvider.GetServices<ICommand>())
        {
            var commandLineApp = command as CommandLineApplication;

            if (commandLineApp == null)
            {
                throw new InvalidCastException("Commands must inherit from ICommand and CommandLineApplication");
            }

            Commands.Add(commandLineApp);
        }
    }
}
```

The constructor takes in the `ServiceProvider`, which is Microsoft's Dependency Injection container. We'll come onto that later. It then uses this to find all the classes that implement `ICommand`, which is an interface we need to register our commands to. I'd prefer to do something like `serviceProvider.GetServices<CommandLineApplication>` but we can't register types under a concrete class and there is no interface provided in `CommandLineUtils` to use. We need to make our own:

```csharp
internal interface ICommand
{
}
```

That's it, only for dependency injection purporses.

## 4. Create a Command
Our `QuoteCommand` class will encapuslate everything about our command and make use of the services that will be injected automatically. It must inherit from `CommandLineApplication` to make use of the CommandLineUtils functionality and it needs our `ICommand` interface so that it can be dependency injected.

```csharp
internal class QuoteCommand : CommandLineApplication, ICommand
{
    private readonly IQuotesService quotesService;

    public QuoteCommand(IQuotesService quotesService)
    {
        this.quotesService = quotesService;
        Name = "quote";
        Description = "gives you a bit of Shakespear";
        HelpOption("-? | -h | --help");
        OnExecute((Func<int>)Execute);
    }

    public int Execute()
    {
        Console.WriteLine(quotesService.Shakespear());
        return 0;
    }
}
```

## 5. Finally - Create the DI container and Command Line App
In Program.cs, remove the Hello World line and then create the dependency injection container:

```csharp
var serviceProvider = new ServiceCollection()
    .AddMyServices()
    .AddSingleton<ICommand, QuoteCommand>()
    .BuildServiceProvider();
```

`AddMyServices` is a [ServiceCollection Extension](DIOnDotNetCoreCLI/DIOnDotNetCoreCLI.Services/ServiceCollectionExtensions.cs) inside my service library. It is responsible for adding its own services to the collection. We then add the `QuoteCommand` on as a Singleton (one per lifetime of the console application) and build it.

We then pass it into our `CommandLineApplicationWithDI` like so:

```csharp
var commandLineApp = new CommandLineApplicationWithDI(serviceProvider);
```

And execute the command with the arguments that come in on the command line.

```csharp
commandLineApp.Execute(args);
```

## Other DI Containers
Microsoft's DI container is really just a set of interfaces and has implementations for other DI containers such as [StructureMap](https://github.com/structuremap/StructureMap.Microsoft.DependencyInjection). With StructureMap, you can add types by convention, so it can automatically find the implementations of `ICommand` for you.

## Further Reading
- [Andrew Lock - Using dependency injection in a .Net Core console application](https://andrewlock.net/using-dependency-injection-in-a-net-core-console-application/). Has a handy section on logging.
- [Tommy Long](http://blog.devbot.net/console-services/). Implements DI at the options level.