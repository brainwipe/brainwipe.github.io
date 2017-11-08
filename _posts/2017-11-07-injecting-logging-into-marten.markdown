---
layout: post
title: "Logging in .NET Core 2 and Marten"
date: 2017-11-07 10:00:00 +01:00
categories: dotnet core marten
---
[Marten](http://jasperfx.github.io/marten/) is a document database that uses Postgres, it's fast and easy to use. I am using it as a projection store. In this post we're going to get the inbuilt Marten logger to use the Microsoft `ILogger` class. I am using Marten 2.3.2 in this blog post.

Marten supports two ways to log output: by listening to events on the store or by implementing `IMartenLogger` or `IMartenSessionLogger` (or both). When using the logger interfaces, you must set them either when the document store is created or when a session is created.

Before I continue, I'd like to give a thanks to [Jeremy D. Miller](https://jeremydmiller.com/) himself for pointing me in this direction. Marten is well laid out, so I had an inkling that this was the right way to go but it was great to have confirmation from the mind behind Marten! Also thanks to [James Hopper](https://github.com/jimgolfgti) for his feedback, I have implemented his improvements.

## Using Dependency Injection
I want to be able to use dependency injection to manage the document session. This is so that for every request on my ASP.NET Core controller I have a single session.

## Prerequisites
I'm going to assume that you have a boilerplate ASP.NET Core 2 application and have installed the [Marten](https://www.nuget.org/packages/Marten/).

## Custom Marten Logger
Before we inject anything, we need a concrete class for Marten to use. Here is the the example Logger from the [Marten Documentation](http://jasperfx.github.io/marten/documentation/documents/diagnostics/) with the Microsoft `ILogger` interface replacing the `Console.WriteLine`. 

```cs
// Adapted from Marten Diagnostics docs: http://jasperfx.github.io/marten/documentation/documents/diagnostics/ 

using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using Marten;
using Marten.Services;
using Microsoft.Extensions.Logging;
using Npgsql;

namespace MyApp
{
    internal class DocumentStoreLogger : IMartenLogger, IMartenSessionLogger
    {
        private readonly ILogger<DocumentStore> logger;

        public DocumentStoreLogger(ILogger<DocumentStore> logger)
        {
            this.logger = logger;
        }

        public IMartenSessionLogger StartSession(IQuerySession session)
        {
            return this;
        }

        public void SchemaChange(string sql)
        {
            logger.LogInformation("Executing DDL change:");
            logger.LogInformation(sql);
            logger.LogInformation();
        }

        public void LogSuccess(NpgsqlCommand command)
        {
            logger.LogInformation(command.CommandText);
            foreach (var p in command.Parameters.OfType<NpgsqlParameter>())
            {
                logger.LogInformation($"  {p.ParameterName}: {p.Value}");
            }
        }

        public void LogFailure(NpgsqlCommand command, Exception ex)
        {
            logger.LogCritical("Postgresql command failed!");
            logger.LogCritical(command.CommandText);
            foreach (var p in command.Parameters.OfType<NpgsqlParameter>())
            {
                logger.LogCritical($"  {p.ParameterName}: {p.Value}");
            }
            logger.LogCritical(ex);
        }

        public void RecordSavedChanges(
            IDocumentSession session, 
            IChangeSet commit)
        {
            var lastCommit = commit;
            logger.LogInformation(
                $"Persisted {lastCommit.Updated.Count()} updates, {lastCommit.Inserted.Count()} inserts, and {lastCommit.Deleted.Count()} deletions");
        }
    }
}
```

## Adding Marten and the Custom Logger to IServiceCollection
Open `Startup.cs` and add two new private methods:

```cs
private static void AddDocumentStore(IServiceCollection services)
{
    services
        .AddSingleton<IMartenLogger, DocumentStoreLogger>()
        .AddSingleton<IDocumentStore>(provider => DocumentStore.For(
        o =>
        {
            o.Logger(provider.GetService<IMartenLogger>());
            o.Connection("Host=localhost;Port=5432;Database=public;User Id=user;Password=password;Pooling=true;Search Path=projections");
            o.AutoCreateSchemaObjects = AutoCreate.All;
            o.DatabaseSchemaName = "projections";
        }))
        .AddScoped<IDocumentSession>(provider => provider.GetService<IDocumentStore>().LightweightSession());
}
```

You can then call `AddDocumentStore(...)` from your `ConfigureServices()` method in `Startup.cs`.

### Explanation
First, we add out custom Marten logger `DocumentStoreLogger` as the IMartenLogger, which will be used for document store level logging. Then we add Document Store as a singleton, giving it the Marten Store logger we just registered. I have hard coded the connection string but I recommend putting that in the `appSettings` configuration ([more](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration?tabs=basicconfiguration)). We're going to let Marten create a new schema in Postgres called `projections`.

Finally, we add a lamda that will construct a Lightweight Session when IDocumentSession is injected. This will log entries on an individual session level. As such, we want this to be Scoped because we want it to be per request.

We are creating a `LightweightSession` because I want to be able to read and write documents and we assign a logger before handing it back.

## Using in a controller
In your controller, you can now inject `IDocumentSession` and make use of it and it will log out to whichever .NET Core 2 Logger providers you have configured.

```cs
[Route("api/[controller]")]
public class ValuesController : Controller
{
    private readonly IDocumentSession session;

    public ValuesController(IDocumentSession session)
    {
        this.session = session;
    }

    // GET api/values
    [HttpGet]
    public IEnumerable<MyEntity> Get()
    {
        return session.Query<MyEntity>();
    }

    // Snipped for brevity
}
```