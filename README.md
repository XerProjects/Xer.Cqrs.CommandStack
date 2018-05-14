# Build

| Branch | Status |
|--------|--------|
| Master | [![Build status](https://ci.appveyor.com/api/projects/status/vembx9v6mvnr24qa/branch/master?svg=true)](https://ci.appveyor.com/project/XerProjects25246/xer-cqrs-commandstack/branch/master) |
| Dev | [![Build status](https://ci.appveyor.com/api/projects/status/vembx9v6mvnr24qa/branch/dev?svg=true)](https://ci.appveyor.com/project/XerProjects25246/xer-cqrs-commandstack/branch/dev) |

# Table of contents
* [Overview](#overview)
* [Features](#features)
* [Installation](#installation)
* [Getting Started](#getting-started)
   * [Command Handling](#command-handling)
      * [Command Handler Registration](#command-handler-registration)
      * [Delegating Commands to Command Handlers](#delegating-commands-to-command-handlers)

# Overview
Simple CQRS library

This project composes of components for implementing the CQRS pattern (Command Handling). This library was built with simplicity, modularity and pluggability in mind.

## Features
* Send commands to registered command handlers.
* Provides simple abstraction for hosted command handlers which can be registered just like an regular command handler.
* Multiple ways of registering command handlers:
    * Simple handler registration (no IoC container).
    * IoC container registration 
      * achieved by creating implementations of IContainerAdapter or using pre-made extensions packages for supported containers:
        * Microsoft.DependencyInjection
        
        [![NuGet](https://img.shields.io/nuget/v/Xer.Cqrs.Extensions.Microsoft.DependencyInjection.svg)](https://www.nuget.org/packages/Xer.Cqrs.Extensions.Microsoft.DependencyInjection/)
        
        * SimpleInjector
        
        [![NuGet](https://img.shields.io/nuget/v/Xer.Cqrs.Extensions.SimpleInjector.svg)](https://www.nuget.org/packages/Xer.Cqrs.Extensions.SimpleInjector/)
        
        * Autofac
        
        [![NuGet](https://img.shields.io/nuget/v/Xer.Cqrs.Extensions.Autofac.svg)](https://www.nuget.org/packages/Xer.Cqrs.Extensions.Autofac/)
        
    * Attribute registration 
      * achieved by marking methods with [CommandHandler] attributes from the Xer.Cqrs.CommandStack.Extensions.Attributes package.
      
      [![NuGet](https://img.shields.io/nuget/v/Xer.Cqrs.Extensions.CommandStack.Attributes.svg)](https://www.nuget.org/packages/Xer.Cqrs.Extensions.CommandStack.Attributes/)

## Installation
You can simply clone this repository, build the source, reference the dll from the project, and code away!

Xer.Cqrs.CommandStack library is available as a Nuget package: 

[![NuGet](https://img.shields.io/nuget/v/Xer.Cqrs.CommandStack.svg)](https://www.nuget.org/packages/Xer.Cqrs.CommandStack/)

To install Nuget packages:
1. Open command prompt
2. Go to project directory
3. Add the packages to the project:
    ```csharp
    dotnet add package Xer.Cqrs.CommandStack
    ```
4. Restore the packages:
    ```csharp
    dotnet restore
    ```

## Getting Started
(Samples are in ASP.NET Core)

### Command Handling

```csharp
// Example command.
public class RegisterProductCommand
{
    public int ProductId { get; }
    public string ProductName { get; }

    public RegisterProductCommand(int productId, string productName) 
    {
        ProductId = productId;
        ProductName = productName;
    }
}

// Command handler.
public class RegisterProductCommandHandler : ICommandAsyncHandler<RegisterProductCommand>
{
    private readonly IProductRepository _productRepository;

    public RegisterProductCommandHandler(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }

    [CommandHandler] // This is in Xer.Cqrs.CommandStack.Extensions.Attributes. This allows the method to registered as a command handler through attribute registration.
    public Task HandleAsync(RegisterProductCommand command, CancellationToken cancellationToken = default(CancellationToken))
    {
        return _productRepository.SaveAsync(new Product(command.ProductId, command.ProductName));
    }
}
```
#### Command Handler Registration

Before we can delegate any commands, first we need to register our command handlers. There are several ways to do this:

##### 1. Simple Registration (No IoC container)
```csharp
// This method gets called by the runtime. Use this method to add services to the container.
public void ConfigureServices(IServiceCollection services)
{            
    ...
    // Repository.
    services.AddSingleton<IProductRepository, InMemoryProductRepository>();

    // Register command delegator.
    services.AddSingleton<CommandDelegator>((serviceProvider) =>
    {
        // Allows registration of a single message handler per message type.
        var registration = new SingleMessageHandlerRegistration();
        registration.RegisterCommandHandler(() => new RegisterProductCommandHandler(serviceProvider.GetRequiredService<IProductRepository>()));

        return new CommandDelegator(registration.BuildMessageHandlerResolver());
    });
    ...
}
```

##### 2. Container Registration
```csharp
// This method gets called by the runtime. Use this method to add services to the container.
public void ConfigureServices(IServiceCollection services)
{            
    ...
    // Repository.
    services.AddSingleton<IProductRepository, InMemoryProductRepository>();

    // Register command handlers to the container. 
    // The AddCqrs extension method is in Xer.Cqrs.Extensions.Microsoft.DependencyInjection package.
    services.AddCqrs(typeof(RegisterProductCommandHandler).Assembly);
    ...
}
```

##### Delegating Commands To Command Handlers
After setting up the command delegator in the IoC container, commands can now be delegated by simply doing:
```csharp
...
private readonly CommandDelegator _commandDelegator;

public ProductsController(CommandDelegator commandDelegator)
{
    _commandDelegator = commandDelegator;
}

// POST api/products
[HttpPost]
public async Task<IActionResult> RegisterProduct([FromBody]RegisterProductCommandDto model)
{
    RegisterProductCommand command = model.ToDomainCommand();
    await _commandDelegator.SendAsync(command);
    return Accepted();
}
...
```
