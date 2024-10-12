### Handler interface 

```csharp
	[Plugin]
	public interface IBusinessRuleResult { }

	[Plugin]
	public interface IBusinessRuleParams { }

	[Plugin]
	public interface IBusinessRule { }

	[Plugin]
	public interface IBusinessRuleHandler<TBusinessRuleType, TBusinessRuleParams>
		where TBusinessRuleParams : IBusinessRuleParams
	{
		Maybe<IBusinessRuleResult> Handler(TBusinessRuleParams prms);
	}
```

### Service 

```csharp
[Export, TransactionScoped]
public class BusinessRulesService : IBusinessRulesService
{
	[Import] public ICompositionRoot Composition { get; set; }
	[Import] public IRuleResolverService RuleResolverService { get; set; }
	[Import] public ILog Log { get; set; }

	/// <inheritdoc/>
	public Maybe<IBusinessRuleResult> ExecuteSingle<TBusinessRuleType, TBusinessRuleParams>(TBusinessRuleParams prms)
		where TBusinessRuleParams : IBusinessRuleParams
			=> Maybe.Unit
				.Then(_ => Composition.Get<IBusinessRuleHandler<TBusinessRuleType, TBusinessRuleParams>>())
				.Then(x => x.Handler(prms));


	/// <inheritdoc/>
	public Maybe<List<IBusinessRuleResult>> ExecuteAll(IEnumerable<IBusinessRuleParams> parameters)
	{
		var results = new List<IBusinessRuleResult>();
		foreach (var prms in parameters)
		{
			Type paramType = prms.GetType();
			Type handlerType = GetHandlerType(paramType);

			if (paramType != null && handlerType != null)
			{
				var instance = RuleResolverService.HandleTypeWrapper((handlerType, paramType))
					.Resolve(Composition, prms)
					.Then(res =>
					{
						results.Add(res);
						return res;
					})
					.WhenError(e => Maybe.Fail<IBusinessRuleResult>(string.Join(", ", e.Messages)))
					.WhenNothingReturnDefault();
				
				//Early return on the first detected failiure
				if (instance.HasValue && instance.Value is BusinessRuleResult r && r.Success == false && r.FailureReason == "business_rule")
					return results;
			}
			else
			{
				var msg = $"No matching Business Rule 'Handler' found for params type {paramType.Name}.";
				Log.Error(msg);
				return Maybe.Fail<List<IBusinessRuleResult>>(msg);
			}
		}
		return results;
	}

	private static Type GetHandlerType(Type paramType)
	=> Assembly.GetExecutingAssembly().GetTypes()
			.FirstOrDefault(t => t.GetInterfaces()
			.Any(i =>
				i.IsGenericType &&
				i.GetGenericTypeDefinition() == typeof(IBusinessRuleHandler<,>) &&
				i.GenericTypeArguments[1] == paramType));
}

[Service]
public interface IRuleResolverService
{
	/// <summary>
	/// Wraps the Erecruit DI to be able to instantiate ER Composition appropirately.
	/// </summary>
	IBusinessRuleTypeWrapper HandleTypeWrapper((Type ResolvedObjectType, Type ResolverParamsType) def);
}

[Service]
public interface IBusinessRuleTypeWrapper
{
	Maybe<IBusinessRuleResult> Resolve(ICompositionRoot composition, object prms);
}

[Export]
class RuleResolverService : IRuleResolverService
{
	static readonly Memoizer Memo = new Memoizer();

	public IBusinessRuleTypeWrapper HandleTypeWrapper((Type ResolvedObjectType, Type ResolverParamsType) def)
		=> GetHandlerTypeWrapper(def);

	static IBusinessRuleTypeWrapper GetHandlerTypeWrapper((Type ResolvedObjectType, Type ResolverParamsType) def)
		=> Memo.Memoize(def, x =>
		{
			var wrapperType = typeof(ResolverWrapper<,>).MakeGenericType(x.ResolvedObjectType, x.ResolverParamsType);
			return (IBusinessRuleTypeWrapper)Activator.CreateInstance(wrapperType);
		});

	class ResolverWrapper<TBusinessRuleType, TBusinessRuleParams> : IBusinessRuleTypeWrapper
		where TBusinessRuleParams : IBusinessRuleParams
	{
		public Maybe<IBusinessRuleResult> Resolve(ICompositionRoot composition, object prms)
		 => composition.Get<IBusinessRuleHandler<TBusinessRuleType, TBusinessRuleParams>>().Handler((TBusinessRuleParams)prms);
	}
}

/// <summary>
/// Contains detailed info about the Business Rules Errors.<br/>
/// A way to opt out of default Maybe Error Handling (initially added for BatchApi use cases). 
/// </summary>
public class BusinessRuleResult : IBusinessRuleResult
{
	public string Message { get; set; }
	/// <summary>
	/// If Success = false => <see cref="FailureReason"/> = "business_rules" tag will be included in the response payload
	/// </summary>
	public string FailureReason { get; set; }
	public bool Success { get; set; }
	public int ErrorCode { get; set; }
}
```


### Config Provider 
```csharp
public interface IBusinessRuleConfigProvider
{
   IEnumerable<TBusinessRule> GetBusinessRules<TBusinessRule>(AboutTypes aboutType)
         where TBusinessRule : IBusinessRule, new();

   IEnumerable<BusinessRuleSettings> GetAllEnabledRules();
}

namespace erecruit.Config
{
	[Export(typeof(IBusinessRuleConfigProvider), Condition = typeof(Composition.Conditions.HasDirectConfigAccess)), TransactionScoped]
	public class ClientSettingsBusinessRuleConfigProvider : IBusinessRuleConfigProvider
	{
		[Import] public IConfigurationService ConfigService { get; set; }

		private static readonly Assembly CurrentAssembly = Assembly.GetExecutingAssembly();

		public IEnumerable<TBusinessRule> GetBusinessRules<TBusinessRule>(AboutTypes aboutType)
		where TBusinessRule : IBusinessRule, new()
		{
			var groupSettings = ConfigService.BusinessRulesSettings.BusinessRulesGroups.Where(g => g.AboutType == aboutType).LastOrDefault();
			if (groupSettings == null) return Enumerable.Empty<TBusinessRule>();
			return groupSettings.BusinessRules.SelectMany(r => r.RuleParameters.OfType<TBusinessRule>());
		}

		public IEnumerable<BusinessRuleSettings> GetAllEnabledRules()
		{
			var groupSettings = ConfigService.BusinessRulesSettings.BusinessRulesGroups.LastOrDefault();
			if (groupSettings == null) return Enumerable.Empty<BusinessRuleSettings>();
			return groupSettings.BusinessRules;
		}

		/// <summary>
		/// Dynamically map XML attributes to BusinesRules Types using reflection.<br>
		/// Currently used by XML parser = ConfigurationService.cs
		/// </summary>
		public static List<IBusinessRule> MapRuleParameters(IEnumerable<XElement> rules)
		{
			var parameters = new List<IBusinessRule>();
			foreach (var rule in rules)
			{
				var type = rule.Parent.Parent.Attribute("Type")?.Value;
				if (string.IsNullOrEmpty(type)) throw new Exception($"'Type' attribute is not defined for BusinessRule");

				Type ruleType = CurrentAssembly.GetTypes().FirstOrDefault(t => t.Name == type);
				if (ruleType == null) throw new Exception($"BusinessRule Type: {type} does not have a definition.");

				if (!typeof(IBusinessRule).IsAssignableFrom(ruleType))
					throw new Exception($"Type {type} does not implement IBusinessRule");

				var ruleInstance = Activator.CreateInstance(ruleType) as IBusinessRule;

				foreach (var property in ruleType.GetProperties())
				{
					var xmlAttrValue = rule.Attribute(property.Name);
					if (xmlAttrValue != null)
					{
						var convertedValue = ConvertToNullableType(xmlAttrValue.Value, property.PropertyType);
						property.SetValue(ruleInstance, convertedValue);
					}
				}
				parameters.Add(ruleInstance);
			}
			return parameters;
		}

		/// <summary>
		/// Handles nullable type conversions.<br/>
		/// Since the 'types' are comming from XML and can have value=""
		/// </summary>
		private static object ConvertToNullableType(string xmlAttrValue, Type targetType)
		{
			var underlyingType = Nullable.GetUnderlyingType(targetType) ?? targetType;

			// Handle XML empty string as null for non-string types
			if (string.IsNullOrEmpty(xmlAttrValue) && underlyingType != typeof(string))
			{
				return null;
			}

			//Handle List<int> (and other List<T> types)
			if (targetType.IsGenericType && targetType.GetGenericTypeDefinition() == typeof(List<>))
			{
				var elementType = targetType.GetGenericArguments()[0]; //e.g. int for List<int>
				if (elementType == typeof(int))
				{
					return xmlAttrValue.Split(new[] { ',' }, StringSplitOptions.RemoveEmptyEntries)
											.Select(int.Parse)
											.ToList();
				}
			}
			return Convert.ChangeType(xmlAttrValue, underlyingType);
		}
	}
}
```


### Usage 
```csharp
Maybe<BusinessRuleResult> ValidateBusinessRules(CancelShiftBusinessRulesPrms prms)
   => BusinessRulesService.ExecuteSingle<LastMinuteActionPreventionForCanceling, LastMinuteActionPreventionForCancelingPrms>(
            new LastMinuteActionPreventionForCancelingPrms
            {
               CandidateId = prms.Candidate.CandidateId,
               ShiftId = prms.Prms.ShiftId
            }).Then(x => x as BusinessRuleResult);

// For multiple registrations we would do 
Maybe<BusinessRuleResult> ValidateAllActiveBusinessRules(BlockingLogicParams prms)
=> BusinessRulesService.ExecuteAll(
   new IBusinessRuleParams[] {
         new IndecisivePreventionPrms
         {
            CandidateId = prms.Ctx.Candidate.CandidateId,
            CompanyId = prms.Prms.ShiftGroupKey.CompanyId,
            Start = prms.Prms.ShiftGroupKey.Start
         },
         new SideJobPreventionPrms
         {
            CandidateId = prms.Ctx.Candidate.CandidateId,
            CompanyId = prms.Prms.ShiftGroupKey.CompanyId,
            Start = prms.Prms.ShiftGroupKey.Start,
            End = prms.Prms.ShiftGroupKey.End,
         },
         new LastMinuteActionPreventionForBookingPrms
         {
            CandidateId = prms.Ctx.Candidate.CandidateId,
            CompanyId = prms.Prms.ShiftGroupKey.CompanyId,
            Start = prms.Prms.ShiftGroupKey.Start
         }
   })
   .Then(x => x?.Cast<BusinessRuleResult>().FirstOrDefault(res => res.Success == false));
```