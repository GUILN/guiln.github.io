---
title: C# Dependency Injection in AWS Lambda Functions
date: "2021-06-02T22:40:32.169Z"
template: "post"
draft: false
slug: "c-sharp-dependency-injection-in-aws-lambda-functions"
category: "C#"
tags:
  - "AWS"
  - "Lambda"
  - "Serverless"
  - "C#"
description: "An approach on how to work with dotnet core DI in AWS Lambda environment."
---

- [Context](#context)
- [Dependency Inversion Principle](#dependency-inversion-principle)
- [Writing Some Code](#writing-some-code)
- [Conclusion](#conclusion)

An Intro about Dependency injection in Lambda environment

## Context

When working with .Net core AWS lambda functions, we do not have some fancy hooks that ables us to wire the dependencies in runtime.
This situation leaves us by ourselves to add the necessary code that overcomes this situation.

## Dependency Inversion Principle

Despite the fact we won't have option to use dotnet core's `IoC Container` in it entire functionality, is not complicated to keep the dependency inversion or injection principle, at the end of the day the most important to keep from this principle is to make sure that we can inject the dependencies and keep our lambda as flexible through different environments and as testable as it would be when working in a "pure" dotnet core environment.
So, with that in mind we will go through a simple, yet efficient approach. Which reminds me the words of *Gary McLean Hall* in book *Adaptive Code Via C#: Agile coding with design patterns and SOLID principles*: 
> When something is so simple, yet so important, people tend to overcomplicate it. DI is no exception...
>
— Gary McLean Hall

So, I'll present in next few lines the most fancy `Poor Man's` DI that I've ever written. Let's write, and discuss some code focusing in:
- Keeping the DI (Most important)
- Avoiding DI anti patterns (`Service Locator`)
- Making it configurable through environments
- Making it testable!
- Keeping it simple

## Writing Some Code

Let's start with `IConfigurationService` interface. This interface will be used as a high level container for the overall configuration dependencies.
```csharp
public interface IConfigurationService
{
     IConfiguration GetConfiguration();
}
```

So `IConfigurationService` will be implemented for each environment, in our case we will only have `Dev` and `local`:

```csharp
//Dev
public class DevConfigurationService : IConfigurationService
{
    public IConfiguration GetConfiguration() => new ConfigurationBuilder()
            .AddSystemsManager("/application-config/order-info-config/")
            .Build();
}
//Local
public class LocalConfigurationService : IConfigurationService
{
    public IConfiguration GetConfiguration() => new ConfigurationBuilder()
            .AddJsonFile("settings.json")
            .Build();
}
```

`IEnvironmentServiceConfiguration` will be the interface to be implemented for every environment to wire all services. If we wish to add some different `ILogger` implementation, for example, this is where we should go. Note that I opted to make the environment selection through a static `Factory Method` in the interface:

```csharp
public interface IEnvironmentServiceConfiguration
{
   public static IEnvironmentServiceConfiguration CreateServiceConfiguration(string environmentName)
   {
           return environmentName switch
           {
               "dev" => new DevServiceConfiguration(),
               "local" => new LocalServiceConfiguration(),
               _ => new LocalServiceConfiguration()
           };
   }
       
   void ConfigureServices(IServiceCollection serviceCollection);
}
```

Following, the implementation of the interface above for both `Dev` and `Local` environments.

```csharp
//Dev
public class DevServiceConfiguration : IEnvironmentServiceConfiguration
{
    public void ConfigureServices(IServiceCollection serviceCollection)
    {
        serviceCollection.AddSingleton<IConfigurationService, DevConfigurationService>();
    }
}
//Local
public class LocalServiceConfiguration : IEnvironmentServiceConfiguration
{
    public void ConfigureServices(IServiceCollection serviceCollection)
    {
        serviceCollection.AddSingleton<IConfigurationService, LocalConfigurationService>();
    }
}
```

To glue all the DI logic and make it automatically change for each environment, we add a singleton class `Bootstrapper` to wire the environment instantiation logic. Note that in that case we are assuming that expected `"local"` and `"dev"` values will come from `ENVIRONMENT` environment variable. This will allow us to configure different `ServiceCollections` among the different environments. We are also using previously declared factory method to instantiate the implementation of `IEnvironmentServiceConfiguration`, which will be used to configure our service collection.


```csharp
public class Bootstrapper
{
        public IServiceProvider ServiceProvider => _serviceCollection.BuildServiceProvider();
        
        private static IServiceCollection _serviceCollection;
        private static Bootstrapper _instance;

        private Bootstrapper()
        {
            ConfigureServices();
        }

        private void ConfigureServices()
        {
            _serviceCollection = new ServiceCollection();
            var environment = Environment.GetEnvironmentVariable("ENVIRONMENT") ??
                              throw new ApplicationException("'ENVIRONMENT' must be set");
            var environmentServiceConfiguration =
                IEnvironmentServiceConfiguration.CreateServiceConfiguration(environment);
            
            environmentServiceConfiguration.ConfigureServices(_serviceCollection);
        }

        public static Bootstrapper GetInstance()
        {
            if (_instance == null)
                _instance = new Bootstrapper();
            return _instance;
        }
}
```

At this point is important to say that, you and your team could easily fall in the temptation of practicing the `Service Locator` anti pattern, this assumption is specially true when using an "home made" IoC container "resolver" because it simply does not feels weird to use your own piece of code to resolve dependencies at any part of your project as it does when dealing with third party IoC container.  
If you fall in this temptation your project will quickly be with "magic" dependencies that will get hard to debug, and hard to tell what does a method / class needs to work just by looking at it usage. So keep the DI spirit downstream your classes / methods!
A good metric that I use to avoid this anti pattern is:

> Whenever I see the usage of a method / class, I should be able to tell all the dependencies by simply looking at method / object's constructor parameters. If I'm not able to do that, some magic stuff is going inside that class. You now have "skyhooks" instead of cranes in your code base.
>
— Just me quoting myself

The usage in code should be simple, as we assume that DI should always be done in the entry point of our program, the entry point for a lambda function will be the constructor of the `Function` class.

```csharp
public class Function
{
        private readonly IConfiguration _configurationService;

        public Function() 
          : this(Bootstrapper.GetInstance()
                            .ServiceProvider
                            .GetService<IConfigurationService>()
                            .GetConfiguration()) {}

        public Function(IConfiguration configurationService) => 
                      (_configurationService) = (configurationService);


        public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest apigProxyEvent,
            ILambdaContext context)
        {
            Console.WriteLine($"Configured value: {_configurationService["configured-variable-per-environment"]}");
            ....
        }
}
```

The idea of having two constructors for the lambda is that the parameterless one will be called by lambda mechanism itself when you, for example, call `sam local invoke` locally, or when you run your lambda in your `AWS` environment. The constructor that receives `IConfiguration` parameter will be used for testing purposes, so we can easily mock `IConfigurationService` dependency for automated tests. 


## Conclusion

This approach keeps DI in lambda with the simplicity of using the `IServiceProvider` and `IServiceCollection` from [Microsoft.Extensions.DependencyInjection][Extensions.DependencyInjection-url] package to register our dependencies. It also overcomes the lack of a mechanism to wire dependencies between different environments that lambda environments leaves us with.

[Extensions.DependencyInjection-url]: https://www.nuget.org/packages/Microsoft.Extensions.DependencyInjection/