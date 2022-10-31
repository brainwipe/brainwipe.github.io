---

title: "Using Brighter on ASP.NET Core 2.0 API"
date: 2017-10-04
categories: asp.net core brighter cqrs
---
In this post you will learn how to get a simple ASP.NET Core 2 API running that uses the [Brighter](https://github.com/BrighterCommand/Brighter) command pattern for sending updates to an in-memory database. Brighter is a library that sits between the API layer and your business logic (and persistance/database layer). This is not production code but the smallest number of steps to get a simple Brighter Command Processor working in ASP.NET Core 2.

You can get the completed code from my example [BrighterOnAspNetCoreExample](https://github.com/brainwipe/BrighterOnAspNetCoreExample) repository.

## Pre-requisites
- Visual Studio 2017

## Command, Query Responsibility Separation
CQRS is an architecture pattern where you use a different model to update than you do to query. [Martin Fowler](https://martinfowler.com/bliki/CQRS.html) does an excellent job of explaining it in detail. Brigher only deals with the Command side, that is the create, update and delete.

## 1. Create ASP.NET Core 2.0 API Project
- In Visual Studio, File &rarr; New Project &rarr; ASP.NET Core
- In the next window choose Web API
- I'm going to call the project BrighterOnAspNetCore
- In the Solution Explorer, right click BrighterOnAspNetCore &rarr; Properties
- In Application Tab, select Target Framework as .NET Core 2.0 and save.

## 2. Update existing dependencies
- Right click on the BrighterOnAspNetCore &rarr; Manage Nuget Packages
- Update any existing packages to v2.  

## 3. Add Brighter AspNet Core dependency
- Still in the Nuget Package Manager window, click "Include Prerelease"
- Install `Paramore.Brighter.AspNetCore` by Daniel Stockhammer. (Brighter will be installed too)

## 4. Create a library to hold your domain
This step is not absolutely necessary but will give a better feel of what a solution ready for production would look like.

- In Solution Explorer, right click the Solution &rarr; Add
- Choose an ASP.NET Core class library, call it Domain.
- Delete the `Class1.cs` that is created for free.
- Add the `Paramore.Brighter` nuget package to the Domain project.
- Add a project reference to the Domain project from the BrighterOnAspNetCore project.

## 5. Add Command and Handler to your domain
A Command is a class that holds all the information needed to make a change to your data model. A Handler performs the actual change to the data model. In this case we're going to create a Command that creates a `Value`, which is simply an integer that you might put into a database.

- In your Domain project, add two folders one called `Commands` and one called `Handlers`
- In `Domain/Commands` add a new class `CreateValueCommand`. 
- It must implement the `IRequest` interface, which is simply an Id for an instance of the command.
- Add an `Email` string property and set it in the constructor.

```csharp
using System;
using Paramore.Brighter;

namespace Domain.Commands
{
    public class CreateValueCommand : IRequest
    {
        public CreateValueCommand(string value)
        {
            Id = Guid.NewGuid();
            Value = value;
        }

        public string Value { get; }

        public Guid Id { get; set; }
    }
}
```

- In `Domain/Handers` add a new class `CreateValueCommandHandler`.
- `CreateValueCommandHandler` must extend the base class `RequestHandler<CreateValueCommand>`. The generic parameter of `CreateValueCommand` is how we say that this handler is for the `CreateValueCommand`.
- We override the `Handle()` method, which is where we would put our database update logic.
- For now, we're going to simply write to the debug console in Visual Studio.

```csharp
using System.Diagnostics;
using Domain.Commands;
using Paramore.Brighter;

namespace Domain.Handlers
{
    public class CreateValueCommandHandler  : RequestHandler<CreateValueCommand>
    {
        public override CreateValueCommand Handle(CreateValueCommand command)
        {
            Debug.WriteLine($"Creating Value {command.Value}");
            return base.Handle(command);
        }
    }
}
```

## 6. Add Brighter configuration to the service collection
ASP.NET Core uses a service collection as a dependency injection container. We need to use this because in our API Controller we want to use the `CommandProcessor` to create commands. We'll come onto that next but first we'll configure ASP.NET to use Brighter.

- Open `BrighterOnAspNetCore/Startup.cs`
- In the `ConfigureServices` method, add the following lines _before_ `.AddMvc()`

```csharp
    services.AddBrighter()
    .HandlersFromAssemblies(typeof(CreateValueCommandHandler).Assembly);
```

This will add all the Handlers in the same assembly as our `CreateValueCommandHandler` to the ASP.NET Core dependency injection container, and perform some default setup. See [Paramore.Brighter.AspNetCore](https://github.com/brainwipe/Paramore.Brighter.AspNetCore) for the options that are available.

## 7. Add Command Processor into our controller and create a command
Commands are all about changing the data in your system, not getting it. 

- Open `BrighterOnAspNetCore/Controllers/ValuesController.cs`
- At the top of the Controller class create the following member variable and constructor:

```csharp
private readonly IAmACommandProcessor commandProcessor;
public ValuesController(IAmACommandProcessor commandProcessor)
{
    this.commandProcessor = commandProcessor;
}
```

The interface `IAmCommandProcessor` will have been registered with the ASP.NET Core dependency injection in the last step. When the controller is instantiated, it will be injected in, ready built and filled with handlers, ready to use.

- In the `Post()` method, create value command and send it to the `commandProcessor`.

```csharp
// POST api/values
[HttpPost("{value}")]
public void Post(string value)
{
    var command = new CreateValueCommand(value);
    commandProcessor.Send(command);
}
```

This is the simplest thing that works; it would be better for the new value we're posting to the service to be in the Body but for that we would need to introduce JSON and a model or a text/plain formatter. For simplicity sake, we'll grab it from the URL instead.

## Testing the service
Run the application and then use as HTTP test client (I like [Postman](https://www.getpostman.com/) but curl would do!) to POST a request to: `http://localhost:58484/api/values/helloworld`, where `localhost:58484` is root of your website and `helloworld` is the data you're sending.

In the Visual Studio Output window, set "Show Output From" to Debug and you should see `Creating value helloworld`. This means that the Command has moved through Brighter and Handled by your command Handler.