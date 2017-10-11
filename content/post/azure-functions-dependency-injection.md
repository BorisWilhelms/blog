+++
Description = "How to implement dependency injection in Azure functions on function level"
Tags = ["Azure", "Functions"]
Keywords = ["Azure", "Functions"]
date = "2017-10-10"
title = "Dependency injection in Azure Functions on function level"
+++

***This solution only supports transient and singleton scoped services. Please see my [follow up post][followup] where you learn how to implement proper dependency injection with support for scoped services.***

Out of the box Azure Functions does not come with dependency injection support you can use in your functions. There are quite a lot of ways to add dependency injection, but most of them rely on the Service Locator (anti-)pattern. In this post you will learn how to implement dependency injection on function level using the extensions API without the Service Locator (anti-)pattern.

# Goal
The goal is to inject dependencies into our functions as a parameters.
```csharp
public static class GreeterFunction
{
    [FunctionName("Greeter")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get")]HttpRequestMessage req,
        [Inject(typeof(IGreeter))]IGreeter greeter)
    {
        return req.CreateResponse(greeter.Greet());
    }
}
```
Here we want to get an implementation of `IGreeter` into our function.

# Inject attribute
First we need to create the `InjectAttribute` which is a binding attribute and receives the type we want to injection.
```csharp
[Binding]
[AttributeUsage(AttributeTargets.Parameter, AllowMultiple = false)]
public class InjectAttribute : Attribute
{
    public InjectAttribute(Type type)
    {
        Type = type;
    }
    public Type Type { get; }
}
```

# Extension Config Provider
As the next and LAST step we need to create an extension config provider, which will be the heart of our dependency injection. 
The provider registers the `InjectAttribute` and handles the resolving of the requested type. 
It is very important that we bind to `dynamic` otherwise we need to create a binding for each type we want to inject.
The `Initialize` method will be called on startup, when the runtime discovers the functions.
Here I use `Microsoft.Extensions.DependencyInjection` but you can use what ever library you like.

```csharp
public class InjectConfiguration : IExtensionConfigProvider
{
    private IServiceProvider _serviceProvider;

    public void Initialize(ExtensionConfigContext context)
    {
        var services = new ServiceCollection();
        RegisterServices(services);
        _serviceProvider = services.BuildServiceProvider(true);

        context
            .AddBindingRule<InjectAttribute>()
            .BindToInput<dynamic>(i => _serviceProvider.GetRequiredService(i.Type));
    }
    private void RegisterServices(IServiceCollection services)
    {
        services.AddSingleton<IGreeter, Greeter>();
    }
}
```

# Conclusion
As you see, it is very easy to add dependency injection on function level using the extension API. 
Besides dependency injection the extension API allows you to create all sorts of input and output bindings!
You can find the full example on [GitHub][github-repo].

[followup]: /post/azure-functions-proper-dependency-injection/
[github-repo]: https://github.com/BorisWilhelms/azure-function-dependency-injection/tree/a8e97b2c487b0b2180181acfe77c68d4e4b002d9