+++
Description = "How to implement proper dependency injection in Azure functions on function level with scoped services!"
Tags = ["Azure", "Functions"]
Keywords = ["Azure", "Functions"]
date = "2017-10-11"
title = "Proper Dependency injection in Azure Functions on function level with scoped services!"
+++
In my [last post][lastpost] you learned how to implement dependency injection in Azure Functions on function level. While this works for services registered as transient or singleton, it does not work with services registered as scoped. In this post you will learn how we can implement proper dependency injection with scopes!

# Inject attribute
We again start with the Inject attribute. Since we are going to change how to resolve bindings, the attribute can now be empty.
```csharp
[Binding]
[AttributeUsage(AttributeTargets.Parameter, AllowMultiple = false)]
public class InjectAttribute : Attribute
{
}
```

# Binding provider
The binding provider is used to create bindings. The method `TryCreateAsync` will be called for each instance of the `InjectAttribute`. 
There we get additional informations, e.g. the parameter on which the attribute is used. Our binding provider also receives the `IServiceProvider` instance which will be passed to the binding.
```csharp
public class InjectBindingProvider : IBindingProvider
{
    private readonly IServiceProvider _serviceProvider;

    public InjectBindingProvider(IServiceProvider serviceProvider) =>
        _serviceProvider = serviceProvider;

    public Task<IBinding> TryCreateAsync(BindingProviderContext context)
    {
        IBinding binding = new InjectBinding(_serviceProvider, context.Parameter.ParameterType);
        return Task.FromResult(binding);
    }
}
```

# Inject binding
The binding class holds the information of the binding. In this case we pass the `IServiceProvider` instance and `Type` the attribute is bound to. The method `BindAsync` will be called, when ever a binding is resolved which happens when a function will be executed. We resolve the service from the `IServiceProvider` and pass it to a `IValueProvider`. 

The `InjectValueProvider` holds the value (the instance of the requested service).
```csharp
public class InjectBinding : IBinding
{
    private readonly Type _type;
    private readonly IServiceProvider _serviceProvider;

    public InjectBinding(IServiceProvider serviceProvider, Type type)
    {
        _type = type;
        _serviceProvider = serviceProvider;
    }

    public bool FromAttribute => true;

    public Task<IValueProvider> BindAsync(object value, ValueBindingContext context) =>
        Task.FromResult((IValueProvider)new InjectValueProvider(value));

    public async Task<IValueProvider> BindAsync(BindingContext context)
    {
        await Task.Yield();
        var value = _serviceProvider.GetRequiredService(_type);
        return await BindAsync(value, context.ValueContext);
    }

    public ParameterDescriptor ToParameterDescriptor() => new ParameterDescriptor();

    private class InjectValueProvider : IValueProvider
    {
        private readonly object _value;

        public InjectValueProvider(object value) => _value = value;

        public Type Type => _value.GetType();

        public Task<object> GetValueAsync() => Task.FromResult(_value);

        public string ToInvokeString() => _value.ToString();
    }
}
```

# Register attribute and the binding provider
Now we glue everything together and register the attribute and binding provider. We again use a `IExtensionConfigProvider` implementation for this.
```csharp
public class InjectConfiguration : IExtensionConfigProvider
{
    public void Initialize(ExtensionConfigContext context)
    {
        var services = new ServiceCollection();
        RegisterServices(services);
        var serviceProvider = services.BuildServiceProvider(true);

        context
            .AddBindingRule<InjectAttribute>()
            .Bind(new InjectBindingProvider(serviceProvider));
    }
    private void RegisterServices(IServiceCollection services)
    {
        services.AddSingleton<IGreeter, Greeter>();
    }
}
```

We now have more a less the same state as in the [last post][lastpost] except, that the attribute got simplified. But this implementation still works only with transient and singleton scoped services.
```
[FunctionName("Greeter")]
public static async Task<HttpResponseMessage> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get")]HttpRequestMessage req,
    [Inject]IGreeter greeter)
{
    return req.CreateResponse(greeter.Greet());
}
```

# Adding support for scoped services
In order to support scoped services we need to change some of the methods. First we need to create and store an `IServiceScope` instance for each function invocation, so that we can share the same scope for the same function invocation. Luckily each function invocation gets a unique id, which will be passed our `InjectBinding`. First we add a `ConcurrentDictionary` to the `InjectBindingProvider` to store the scopes.

```
public class InjectBindingProvider : IBindingProvider
{
    public static readonly ConcurrentDictionary<Guid, IServiceScope> Scopes =
        new ConcurrentDictionary<Guid, IServiceScope>();
    
    ...
}
```

Now we change the `BindAsync` method in the `InjectBinding`. The `BindingContext` contains the FunctionInstanceId which is the unique id, created for each function invocation. So for each invocation we create a new scope and store the scope in the `ConcurrentDictionary`. We then resolve the service from the scope.
```
public async Task<IValueProvider> BindAsync(BindingContext context)
{
    await Task.Yield();

    var scope = InjectBindingProvider.Scopes.GetOrAdd(context.FunctionInstanceId, (_) => _serviceProvider.CreateScope());
    var value = scope.ServiceProvider.GetRequiredService(_type);

    return await BindAsync(value, context.ValueContext);
}
```
At this point we are already able to resolve scoped services, BUT we do not clean up and destroy the scope, so all resolved service will life forever. In order to cleanup the scopes, we implement an `IFunctionInvocationFilter` and `IFunctionExceptionFilter` which are called whenever a function execution ended or when an exception within the function occurred. 
```
public class ScopeCleanupFilter : IFunctionInvocationFilter, IFunctionExceptionFilter
{
    public Task OnExceptionAsync(FunctionExceptionContext exceptionContext, CancellationToken cancellationToken)
    {
        RemoveScope(exceptionContext.FunctionInstanceId);
        return Task.CompletedTask;
    }

    public Task OnExecutedAsync(FunctionExecutedContext executedContext, CancellationToken cancellationToken)
    {
        RemoveScope(executedContext.FunctionInstanceId);
        return Task.CompletedTask;
    }

    public Task OnExecutingAsync(FunctionExecutingContext executingContext, CancellationToken cancellationToken) =>
        Task.CompletedTask;

    private void RemoveScope(Guid id)
    {
        if (InjectBindingProvider.Scopes.TryRemove(id, out var scope))
        {
            scope.Dispose();
        }
    }
}
```
As the last step we need to register the filter. Therefor we change the `Initialize` method of our `InjectConfiguration`.
```
public void Initialize(ExtensionConfigContext context)
{
    ...

    var registry = context.Config.GetService<IExtensionRegistry>();
    var filter = new ScopeCleanupFilter();
    registry.RegisterExtension(typeof(IFunctionInvocationFilter), filter);
    registry.RegisterExtension(typeof(IFunctionExceptionFilter), filter);
}
```
# Conclusion
Although Azure Functions does not come with dependency injection for you functions, it can be added quite easy. It is still a "hacky" solution, but it works :-)
You can find the full example on [GitHub][github-repo].

[lastpost]: /post/azure-functions-dependency-injection/
[github-repo]: https://github.com/BorisWilhelms/azure-function-dependency-injection