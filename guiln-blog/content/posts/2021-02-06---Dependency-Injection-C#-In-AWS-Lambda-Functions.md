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
description: "An approach on how to work with dotnet's DI in AWS Lambda environment."
socialImage: "/media/42-line-bible.jpg"
---

- [The Context](#the-context)
- [Dependency Inversion Principle](#dependency-inversion-principle)
- [Writing Some Code](#writing-some-code)
- [Conclusion](#conclusion)

An Intro about Dependency injection in Lambda environment

## The Context

When working with .Net core AWS lambda functions, we do not have some fancy `IoC containers` that are able to wire the dependencies in runtime.
If we were working in a "classic" .Net core  project we would have some options of IOC containers, although one largely used in great part of the .Net core projects is the built in IServiceCollection interfaces (IServiceCollection will be used here to refer to .Net core out of the box DI mechanism).
As we are not in a full .Net core environment we do not have access to .Net runtime "injectors", and as we are working with AWS lambdas we do not have a way to hook the DI in runtime (at least not at the time that we write this document).
This context leaves us with some options of non standard way of injecting dependency, and so it leaves this decision to be made.

## Dependency Inversion Principle

As previously discussed we won't have option to use dotnet core's `IoC Container`, although is not complicated to keep our `D`from our famous `SOLID`, at the end of the day we only want to make sure that we can inject the dependencies and keep our lambda as flexible through different environments and as testable as it would be when working in a "pure" dotnet core environment. 
One word: `KISS` - Keep it simple, or in the words of *Gary McLean Hall* in book *Adaptative Code Via C#*: "When something is so simple, yet so important, people tend to overcomplicate it. DI is no exception...".

> When something is so simple, yet so important, people tend to overcomplicate it. DI is no exception...
>
— Gary McLean Hall, Adaptative Code Via C#

`Poor Man's` DI is the `KISS` of this principle, for given situation. So let's write some code focusing in:
- Keeping the DI
- Avoiding DI anti patterns (`Service Locator`)
- Making it configurable through environments
- Making it testable!
- Keeping it simple


## Writing Some Code

Let's start with `IConfigurationService` interface.
```csharp
public interface IConfigurationService
{
     IConfiguration GetConfiguration();
}
```

Let's say we have two environments `Dev` and `local`:

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
            .AddSystemsManager("/application-config/order-info-config/")
            .Build();
}
```

`IEnvironmentServiceConfiguration` will be the interface to be implemented for every environment in order to register services in IoC container, opted to make the environment selection through a static `Factory Method` in the interface:

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

Wire `Dev` and `Local` configuration in IOC container, registering the `IConfigurationService` per environment.

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

Singleton class to wire the environment instantiation logic. Note that in that case we are assuming that expected `"local"` and `"dev"` values will come from `ENVIRONMENT` environment variable. This will allow us to configure different `ServiceCollections` among the different environments.


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

At this point is important to say that, you and your team could easily fall in the temptation of practicing the `Service Locator` anti pattern, this assumption is specially true when using an "home made" IoC container resolver because it simply don't feel weird to use your own piece of code to resolve dependencies at any part of your project as it does when dealing with third party IoC container (even if is the one that ships with dotnet core). 
Be aware! If you fall in temptation to use `Service Locator` your project will quickly be with "magic" dependecies that will get hard to debug, and hard to tell what does a method / class needs to work just by looking at it usage. So keep the DI spirit downstream your classes / methods!
A good metric that I use to avoid this anti pattern is:

> Whenever I see the usage of a method / class, I should be able to tell all the dependencies by simply looking at method / object's constructor parameters. If I'm not able to do that, some magic stuff is going inside that class. You now have "skyhooks" instead of cranes in your code base.
>
— Jus me quoting myself

The usage in code should be simple, as we assume that DI should always be done in the entry point of our program, the entry point for a lambda function will be it constructor.

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

The idea of having 2 constructors for the lambda is that the parameterless one will be called by lambda mechanism itself when you, for example, call `sam local invoke` locally, or when you run your lambda in your `AWS` environment. The contructor that receives `IConfiguration` parameter will be used for testing purposes, so we can easily mock `IConfigurationService` dependency in our automated tests. 


## Conclusion

Overall conclusion on the approach
