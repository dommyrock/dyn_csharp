
## Inspired by this reflection based 'Resolver' discovery service form one of the codebases I worked on  

```csharp
	public sealed class ResolverDefinitions
	{
		readonly IDictionary<string, (Type ResolvedObjectType, Type ResolverParamsType)> Definitions;

		public ResolverDefinitions(IDictionary<string, (Type ResolvedObjectType, Type ResolverParamsType)> defs)
			=> Definitions = defs;

		public Maybe<(Type ResolvedObjectType, Type ResolverParamsType)> GetDefinitionFor(string resolvedObjectType)
			=> Definitions.ContainsKey(resolvedObjectType.ToLower())
					? Definitions[resolvedObjectType.ToLower()]
					: Maybe.Nothing<(Type ResolverType, Type ResolverParamsType)>();

		public (Type ResolvedObjectType, Type ResolverParamsType)[] GetAllDefinitions()
			=> Definitions.Values.ToArray();
	}


	public static class ResolverDiscovery
	{
		readonly static Memoizer Memo = new Memoizer();

		public static ResolverDefinitions Resources
			=> Memo.Memoize(nameof(Resources), _ => GetDefinitions(
					resolverFilter: x => x.Namespace.Contains("Resolvers.Resources")
				));

		public static ResolverDefinitions Mutations
			=> Memo.Memoize(nameof(Mutations), _ => GetDefinitions(
					resolverFilter: x => x.Namespace.Contains("Resolvers.Mutations")
				));

      //This being the core layer 
		static ResolverDefinitions GetDefinitions(Func<Type, bool> resolverFilter)
			=> new ResolverDefinitions(
					Assembly.GetExecutingAssembly().GetTypes()
						.Where(t => t.GetCustomAttribute<ExportAttribute>() != null && resolverFilter(t))
						.SelectMany(t => t.GetInterfaces())
						.Where(t => t.IsGenericType && t.GetGenericTypeDefinition() == typeof(IResolver<,>))
						.Select(GetResolverDefinition)
						.ToDictionary(x => x.ResolvedTypeName, x => (x.ResolvedObjectType, x.ResolverParamsType)));

		static (string ResolvedTypeName, Type ResolvedObjectType, Type ResolverParamsType) GetResolverDefinition(Type resolver)
		 => (resolver.GenericTypeArguments[0].Name.ToLower(), resolver.GenericTypeArguments[0], resolver.GenericTypeArguments[1]);
	}
```

---

<br/>

## I wanted to implement something similar for my BusinessRules handler engine


Where U wanted to resolve and call the appropriate handler for a given business rule using reflection. <br>
I looked into taking advantage of the `GetDefinitions` style of dynamic discovery and to invoke the handler based on the type parameters at runtime.<br> 
This approach dynamically inspects available types and calls the correct handler using reflection, without a need for a registry.

### Where registry code would look something like this.

```csharp
public interface IBusinessRuleHandlerRegistry
{
    IBusinessRuleHandler<TBusinessRuleParams> GetHandler<TBusinessRuleParams>()
        where TBusinessRuleParams : IBusinessRuleParams;
}

public class BusinessRuleHandlerRegistry : IBusinessRuleHandlerRegistry
{
    private readonly Dictionary<Type, object> _handlers = new Dictionary<Type, object>();

    public BusinessRuleHandlerRegistry()
    {
        // Register handlers
        Register<LastMinuteActionPreventionForBookingPrms>(new LastMinuteActionPreventionForBookingHandler());
        // Register other handlers here...
    }

    public IBusinessRuleHandler<TBusinessRuleParams> GetHandler<TBusinessRuleParams>()
        where TBusinessRuleParams : IBusinessRuleParams
    {
        var type = typeof(TBusinessRuleParams);
        if (_handlers.TryGetValue(type, out var handler))
        {
            return (IBusinessRuleHandler<TBusinessRuleParams>)handler;
        }

        throw new ArgumentException($"No handler registered for type {type}");
    }

    private void Register<TBusinessRuleParams>(IBusinessRuleHandler<TBusinessRuleParams> handler)
        where TBusinessRuleParams : IBusinessRuleParams
    {
        _handlers[typeof(TBusinessRuleParams)] = handler;
    }
}

```


<br>

I chose bellow approach that uses a bit of reflection to resolve dynamic type checking for `Speciffic type` handlers:

### Step 1: Create the `Handler` Interface

First, ensure that you have the handler interface in place, similar to before:

```csharp
public interface IBusinessRuleHandler<TBusinessRuleParams>
    where TBusinessRuleParams : IBusinessRuleParams
{
    Task<IBusinessRuleResult> HandleAsync(TBusinessRuleParams prms);
}
```

### Step 2: Discover Handlers Using Reflection

You can define a method similar to your `GetDefinitions`, but it will look for all types that implement `IBusinessRuleHandler<T>`, and then match them based on the type of `TBusinessRuleParams`. Here’s an example of how this might look:

```csharp
public static class HandlerResolver
{
    public static Dictionary<Type, Type> GetHandlerDefinitions()
    {
        return Assembly.GetExecutingAssembly().GetTypes()
            .Where(t => !t.IsAbstract && !t.IsInterface)
            .SelectMany(t => t.GetInterfaces(), (type, @interface) => new { type, @interface })
            .Where(t => t.@interface.IsGenericType && t.@interface.GetGenericTypeDefinition() == typeof(IBusinessRuleHandler<>))
            .ToDictionary(
                t => t.@interface.GetGenericArguments()[0], // Extract the parameter type (TBusinessRuleParams)
                t => t.type);  // Store the concrete handler class
    }
}
```

### Step 3: Use Reflection to Invoke the Correct Handler

In your `BusinessRuleService`, you can use the `GetHandlerDefinitions` method to map parameter types to handler types and invoke the correct handler using reflection. Here’s how to refactor `ExecuteAsync` to use reflection:

```csharp
public class BusinessRuleService
{
    private readonly Dictionary<Type, Type> _handlerDefinitions;

    public BusinessRuleService()
    {
        _handlerDefinitions = HandlerResolver.GetHandlerDefinitions();
    }

    public async Task<IBusinessRuleResult> ExecuteAsync<TBusinessRuleParams>(TBusinessRuleParams prms)
        where TBusinessRuleParams : IBusinessRuleParams
    {
        var paramType = typeof(TBusinessRuleParams);

        if (_handlerDefinitions.TryGetValue(paramType, out var handlerType))
        {
            // Create an instance of the handler
            var handlerInstance = Activator.CreateInstance(handlerType);
            
            // Find the HandleAsync method
            var handleMethod = handlerType.GetMethod("HandleAsync", new[] { paramType });

            if (handleMethod != null)
            {
                // Invoke HandleAsync using reflection
                var resultTask = (Task<IBusinessRuleResult>)handleMethod.Invoke(handlerInstance, new object[] { prms });
                return await resultTask;
            }
        }

        throw new ArgumentException($"No handler found for type {paramType.Name}");
    }
}
```

### Step 4: Example Handler Implementation

Here's an example implementation of a specific handler:

```csharp
public class LastMinuteActionPreventionForBookingHandler : IBusinessRuleHandler<LastMinuteActionPreventionForBookingPrms>
{
    public async Task<IBusinessRuleResult> HandleAsync(LastMinuteActionPreventionForBookingPrms prms)
    {
        var bookingSideJobPreventionRules = BusinessRuleConfigProvider
            .GetBusinessRules<LastMinuteActionPreventionForBooking>(AboutTypes.Shift)
            .Where(r => r.Enforce.HasValue && r.Enforce == true)
            .ToList();

        var msg = "";
        // Continue business logic...
        return new BusinessRuleResult { Success = true, Message = msg };
    }
}
```

### Step 5: Example Usage

Now, you can use the `BusinessRuleService` to resolve and execute the correct handler dynamically:

```csharp
var businessRuleService = new BusinessRuleService();

var ruleParams = new LastMinuteActionPreventionForBookingPrms
{
    CandidateId = 1,
    CompanyId = 2,
    Start = DateTime.Now
};

var result = await businessRuleService.ExecuteAsync(ruleParams);
```

### Explanation

1. **Reflection-based Handler Discovery**: The `GetHandlerDefinitions` method uses reflection to scan all types in the assembly, looking for those that implement `IBusinessRuleHandler<T>`. It builds a dictionary where the key is the `TBusinessRuleParams` type, and the value is the concrete handler type.

2. **Dynamic Handler Invocation**: In `ExecuteAsync`, the appropriate handler type is found using the dictionary. The handler is instantiated using `Activator.CreateInstance`, and then its `HandleAsync` method is called using reflection.

3. **Flexibility**: This approach avoids using a manual registry or hardcoding specific handlers in `ExecuteAsync`. New handlers can be added simply by implementing the `IBusinessRuleHandler<T>` interface for new types, and the reflection mechanism will automatically discover them.

### Performance Consideration

Reflection can have a performance overhead, especially if used frequently in a tight loop. To mitigate this, you could cache the `MethodInfo` for the handler methods after the first lookup to avoid the cost of `GetMethod` in subsequent calls.

This approach, while a bit more complex due to reflection, provides flexibility in dynamically discovering and invoking handlers for business rules based on their parameter types.