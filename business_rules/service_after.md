# New design 

Main change was moving handler interface to only One params type.<br/>
We could do this because we figured that It would be Unique enough to indentify each type passed into a generic list.<br/>

This also enabled us to remove reflection from the Service code and to streamline the service layer api.

```csharp
[Plugin]
public interface IBusinessRuleHandler<TBusinessRuleParams>
	where TBusinessRuleParams : IBusinessRuleParams
{
	Maybe<IBusinessRuleResult> Handler(TBusinessRuleParams prms);
}
```

Note: If we kept reflection solution , it would still work with few changes like this.
```csharp
//change 1
public Maybe<IBusinessRuleResult> ExecuteSingle<TBusinessRuleParams>(TBusinessRuleParams prms)
where TBusinessRuleParams : IBusinessRuleParams
	=> Maybe.Unit
		.Then(_ => Composition.Get<IBusinessRuleHandler<TBusinessRuleParams>>())
		.Then(x => x.Handler(prms));

//change 2
private static Type GetHandlerType(Type paramType)
=> Assembly.GetExecutingAssembly().GetTypes()
	.FirstOrDefault(t => t.GetInterfaces()
	.Any(i =>
		i.IsGenericType &&
		i.GetGenericTypeDefinition() == typeof(IBusinessRuleHandler<>) &&
		i.GenericTypeArguments[1] == paramType));

//change 3
class ResolverWrapper<TBusinessRuleType, TBusinessRuleParams> : IBusinessRuleTypeWrapper
where TBusinessRuleParams : IBusinessRuleParams
{
	public Maybe<IBusinessRuleResult> Resolve(ICompositionRoot composition, object prms)
		=> composition.Get<IBusinessRuleHandler<TBusinessRuleParams>>().Handler((TBusinessRuleParams)prms);
}

//Usage Single
Maybe<BusinessRuleResult> ValidateBusinessRules(CancelShiftBusinessRulesPrms prms)
	=> BusinessRulesService.ExecuteSingle(
		new LastMinuteActionPreventionForCancelingPrms
		{
			CandidateId = prms.Candidate.CandidateId,
			ShiftId = prms.Prms.ShiftId
		}).Then(x => x as BusinessRuleResult);

//Usage Multiple Rule config
Maybe<BusinessRuleResult> ValidateAllActiveBusinessRules(BlockingLogicParams prms)
=> new IBusinessRuleParams[] {
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
}.Select(BusinessRulesService.ExecuteSingle)
.AsMaybe()
.Then(x => x?.Cast<BusinessRuleResult>().FirstOrDefault(res => res.Success == false));
```

### One issue that I encountered was that .Cast(T) couldn't infer the concrete type 'BusinessRuleResult' from IBusinessRuleResult and threw Excepiton

```csharp
//'Unable to cast object of type '_Error [erecruit.Config.IBusinessRuleResult]' to type'erecruit.BL.BusinessRuleResult'
Maybe<BusinessRuleResult> ValidateAllActiveBusinessRules(BlockingLogicParams prms)
=> new IBusinessRuleParams[] {
			//...
	}.Select(BusinessRulesService.ExecuteSingle)
	.Lift()
	//Fix was to use OfType<T>
	.Then(x => x?.OfType<BusinessRuleResult>().FirstOrDefault(res => res.Success == false));
```


### Service becomes compact since we can remove type reflection code

```csharp
[Export, TransactionScoped]
public class BusinessRulesService : IBusinessRulesService
{
	[Import] public ICompositionRoot Composition { get; set; }
	[Import] public ILog Log { get; set; }

	/// <inheritdoc/>
	public Maybe<IBusinessRuleResult> ExecuteSingle<TBusinessRuleParams>(TBusinessRuleParams prms)
	where TBusinessRuleParams : IBusinessRuleParams
		=> Maybe.Unit
			.Then(_ => Composition.Get<IBusinessRuleHandler<TBusinessRuleParams>>())
			.Then(x => x.Handler(prms));
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
````
