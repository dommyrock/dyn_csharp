# dyn_c# - Dynamic type inference / allocations in c# 

[Using type dynamic - msdn](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/interop/using-type-dynamic)
[dyn dispatch -> Visitor pattern](https://chodounsky.com/2014/01/29/dynamic-dispatch-in-c-number/)

#### Use Case intro

I had a case where I wanted to have a 'marker' $${\color{orange}IBusinessRuleParameters}$$ which I could use at Runtime to infer the concrete type + params.
```csharp
public interface IBusinessRule<TParameters> where TParameters : IBusinessRuleParameters, new()
{
  TParameters RuleParameters { get; }
}
```
While the concrete classes looked like this
```csharp
public class BusinessRulesDefinitions
{
	public class IndecisivePreventionRuleParameters : IBusinessRuleParameters
	{
		public IndecisivePreventionRuleParameters() { }
		public int? ForCandidateStatusId { get; set; }
		public List<int> IfShiftEndReasonIds { get; set; }
		public int? ForTheNextXDays { get; set; }
		public int? RoleTypeId { get; set; }
		public bool? Enforce { get; set; }
	}

	public class SideJobPreventionRuleParameters : IBusinessRuleParameters
	{
		public SideJobPreventionRuleParameters() { }
		public int? ForCandidateStatusId { get; set; }
		public int? RoleTypeId { get; set; }
		public bool? Enforce { get; set; }
	}

	public class LastMinuteActionPreventionRuleParameters : IBusinessRuleParameters
	{
		public LastMinuteActionPreventionRuleParameters() { }
		public List<int> ForCandidateStatusIds { get; set; }
		public bool? Enforce { get; set; }
		public int? Minutes { get; set; }
	}
//...
}
```
<br/>

---

## Final design ended up being fully *dynamic* approach , where types are infered at runtime
// TOODO : UPDATE section


<br>

---

## Some other possible options I was exploring

The requirement for a **parameterless constructor** in this generic composition is directly related to how the `new()` constraint is used in C#. When working with generics and needing to instantiate an object of a generic type, the `new()` constraint ensures that the compiler knows the type has a parameterless constructor so that it can create an instance of the type at runtime via `new T()`.

Here’s a breakdown of why this is necessary and other approaches you can take to map concrete type properties without needing a parameterless constructor:

### Why You Need a Parameterless Constructor

1. **`new T()` Needs a Parameterless Constructor**:
    - When you call `new TParameters()`, C# needs to know that the generic type `TParameters` has a parameterless constructor.
    - The `new()` constraint ensures that the generic type can be instantiated via `new T()` without needing arguments. If the type doesn't have this constructor, the `new T()` operation would fail.
   
2. **Activator.CreateInstance**:
    - If you use `Activator.CreateInstance(typeof(T))`, it internally tries to find a parameterless constructor to create an instance of `T`. That's why even `Activator.CreateInstance` requires a `new()` constraint or some other way to pass arguments.

3. **Constraints on Generics**:
    - C# generics are compiled with some safety features in mind, and when you want to instantiate a type via generics, the compiler needs a guarantee that a type can be instantiated safely (without needing complex constructor logic or arguments).
    - The `new()` constraint guarantees this at compile-time.

### Alternative Approaches Without a Parameterless Constructor

If you want to avoid the constraint of requiring a parameterless constructor for your types, you can use other approaches to instantiate objects dynamically. These methods give you more flexibility in how you instantiate and map properties onto your types at runtime.

#### 1. **Using `Activator.CreateInstance` with Constructor Parameters**

You can instantiate objects by passing arguments to the constructor using `Activator.CreateInstance`. This allows you to create objects even if they don't have a parameterless constructor.

```csharp
public TParameters CreateInstanceWithParameters<TParameters>(params object[] args)
{
    return (TParameters)Activator.CreateInstance(typeof(TParameters), args);
}
```

This method allows you to pass any number of arguments to the constructor of `TParameters`. For example, if `TParameters` has a constructor that takes specific arguments, you can provide those dynamically.

#### 2. **Using Reflection to Set Properties (Without Constructor)**

If your main goal is to map properties dynamically (like you are doing with `ConvertToParameters`), you can instantiate the object without needing a constructor and then populate the properties using reflection.

You can use `FormatterServices.GetUninitializedObject()` to create an object without invoking any constructor:

```csharp
public TParameters ConvertToParametersWithoutConstructor<TParameters>(Dictionary<string, object> parameters)
    where TParameters : IBusinessRuleParameters
{
    var paramInstance = (TParameters)FormatterServices.GetUninitializedObject(typeof(TParameters));

    foreach (var prop in typeof(TParameters).GetProperties())
    {
        if (parameters.TryGetValue(prop.Name, out var value))
        {
            prop.SetValue(paramInstance, Convert.ChangeType(value, prop.PropertyType));
        }
    }

    return paramInstance;
}
```

`FormatterServices.GetUninitializedObject()` creates an instance of the class without calling any constructor, which can be useful if your types don’t have parameterless constructors.

> **Warning**: This approach bypasses normal object initialization, so it could lead to incomplete or invalid states if your type’s constructor is responsible for important initialization logic. Use it carefully!

#### 3. **Factory Method Pattern**

If you don't want to rely on reflection and dynamic instantiation, another approach is to use the **Factory Method Pattern**. In this pattern, you create a factory responsible for creating instances of your rules and their parameters.

For example:

```csharp
public interface IBusinessRuleFactory
{
    IBusinessRule<TParameters> CreateBusinessRule<TParameters>(Type ruleType);
    TParameters CreateParameters<TParameters>(Dictionary<string, object> parameters);
}

public class BusinessRuleFactory : IBusinessRuleFactory
{
    public IBusinessRule<TParameters> CreateBusinessRule<TParameters>(Type ruleType)
    {
        // Use Activator or custom logic to instantiate the business rule
        return (IBusinessRule<TParameters>)Activator.CreateInstance(ruleType);
    }

    public TParameters CreateParameters<TParameters>(Dictionary<string, object> parameters)
    {
        var paramInstance = new TParameters();
        foreach (var prop in typeof(TParameters).GetProperties())
        {
            if (parameters.TryGetValue(prop.Name, out var value))
            {
                prop.SetValue(paramInstance, Convert.ChangeType(value, prop.PropertyType));
            }
        }
        return paramInstance;
    }
}
```

This allows you to have more control over the instantiation logic, and you can easily swap out or modify how objects are created without relying on `new()`.

#### 4. **Dependency Injection (DI) with Factories**

If you are using a dependency injection (DI) container, you can also leverage factories and DI to avoid needing a parameterless constructor. With DI, the container can manage the creation of objects and automatically inject required dependencies.

```csharp
public class BusinessRuleFactory : IBusinessRuleFactory
{
    private readonly IServiceProvider _serviceProvider;

    public BusinessRuleFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IBusinessRule<TParameters> CreateBusinessRule<TParameters>(Type ruleType)
    {
        return (IBusinessRule<TParameters>)_serviceProvider.GetService(ruleType);
    }

    public TParameters CreateParameters<TParameters>(Dictionary<string, object> parameters)
    {
        // Similar mapping logic for parameters
    }
}
```

This way, the DI container manages object instantiation without requiring a parameterless constructor.

### To sum up all alternatives

- The **parameterless constructor (`new()`) constraint** is required when you want to use `new T()` or `Activator.CreateInstance` in a generic context to ensure the object can be instantiated.
- **Alternatives**:
  - **`Activator.CreateInstance` with parameters**: Allows instantiation even for types that don't have a parameterless constructor.
  - **Reflection with `FormatterServices.GetUninitializedObject`**: Lets you instantiate without invoking a constructor at all.
  - **Factory Method Pattern**: Encapsulates the instantiation logic, offering more flexibility.
  - **Dependency Injection**: Leverages DI containers to handle object creation, often avoiding the need for explicit constructors.

> If your objects have complex constructors or require dependencies, using a factory or DI might be the best solution. <br/>
> For simpler scenarios, `new()` with generics works well and is easy to implement.
