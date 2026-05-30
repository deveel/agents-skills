# Entity Manager Lifecycle Reference

Use this reference when the agent needs more concrete guidance on how to explain
`Entity Manager` responsibilities within a DDD-oriented architecture.

## Role of the Entity Manager

The `Entity Manager` is the orchestration boundary for entity lifecycle work. It
should coordinate the lifecycle flow without stealing business behavior from the
aggregate.

Treat `EntityManager<>` as an extension point: projects can override the base
class to implement entity-specific lifecycle behavior while preserving a common
orchestration contract.

Keep this split clear:

- **Aggregate root and entities** own invariants, state transitions, and domain meaning
- **IEntityValidator** implementations provide reusable async validation streams (`IAsyncEnumerable<ValidationResult>`) per entity or aggregate
- **Entity Manager** coordinates lifecycle execution order and update-time validation
- **Repositories** provide aggregate persistence at the correct boundary and do not perform autonomous post-set update orchestration
- **Application services or handlers** translate use cases into lifecycle operations

## Preferred lifecycle flow

When describing an implementation, anchor it on this sequence:

1. Validate command intent and required inputs
2. Create or load the aggregate root
3. Invoke domain behavior that enforces invariants
4. Consume `IEntityValidator` async validation results and lifecycle-specific policy checks
5. Let the `Entity Manager` orchestrate update validation and call repository persistence
6. Trigger post-success integration work only after the state transition is valid

## Lifecycle guidance by operation

### Validation

- Distinguish boundary validation from business validation
- Keep business invariants in the domain model
- Implement dedicated `IEntityValidator` types for reusable entity validation policies
- Use the `Entity Manager` to consume validator result streams consistently before persistence

### Creation

- Prefer explicit constructors, factories, or named creation methods
- Avoid building aggregates through public writable properties
- Use the `Entity Manager` to coordinate creation workflow and persistence

### Mutation

- Replace arbitrary setters with intent-based domain methods
- Route state changes through aggregate behavior instead of service-owned mutation logic
- Use the `Entity Manager` to enforce a consistent mutation workflow
- Keep `IRepository` implementations persistence-focused; they should not update entities directly after set operations

### Deletion

- Treat deletion as a domain policy, not a raw database action
- Decide explicitly whether deletion means hard delete, soft delete, archive, or status transition
- Let the `Entity Manager` coordinate checks and persistence for the chosen policy

## API surface mirroring

The `Entity Manager` should expose a method surface that mirrors the underlying
repository. For example, if the `IRepository` implementation provides `FindUserByNameAsync`,
the `EntityManager<>` subclass should also expose `FindUserByNameAsync`. This
keeps the application layer from taking a direct dependency on the repository and
routes all access through the managed lifecycle boundary.

The three rules that differentiate the Entity Manager method from its repository
counterpart:

1. **No `CancellationToken` parameter.** The Entity Manager obtains the current
   cancellation token through an injected context service (e.g. an
   `IOperationContext` or equivalent ambient-context abstraction). Callers do not
   pass tokens directly; the manager resolves them internally.

2. **Always returns `OperationResult` or `OperationResult<TEntity>`.** Every
   method on the Entity Manager communicates outcome through a typed result
   object, never by throwing or by returning a raw entity or `bool`. This
   includes query-style methods — a lookup that finds nothing returns a
   failed or empty `OperationResult<T>`, not `null`.

3. **Repository errors are caught and wrapped.** The Entity Manager method
   wraps every call to the underlying `IRepository` in a try/catch (or equivalent
   error-handling policy). Exceptions originating in the repository are converted
   to `OperationResult.Fail(...)` with an appropriate error description, so callers
   always receive a structured result instead of an unhandled exception propagating
   through the application layer.

### Guidance for generating Entity Manager members

When generating or reviewing an `EntityManager<>` subclass:

- For every repository method that the application layer needs to call, add a
  matching method on the manager with the same name and the same non-cancellation
  parameters.
- Remove `CancellationToken` from the signature; resolve it from the injected
  context service inside the method body.
- Change the return type to `Task<OperationResult>` or
  `Task<OperationResult<TEntity>>` (or their synchronous equivalents if the
  framework supports them).
- Wrap the repository call in a try/catch and return `OperationResult.Fail` for
  any exception, logging if appropriate before returning.
- Keep business-rule checks (validators, invariant enforcement) before the
  repository call so that structural failures are reported before persistence
  is attempted.

### Example shape

```csharp
// Repository contract
Task<User?> FindUserByNameAsync(string name, CancellationToken cancellationToken);

// Entity Manager counterpart
public async Task<OperationResult<User>> FindUserByNameAsync(string name)
{
    try
    {
        // CaancellationToken is a property on the manager, resolved from the context service
        var user = await _repository.FindUserByNameAsync(name, CancellationToken);
        return user is null
            ? OperationResult.Fail<User>("USER_NOT_FOUND", "User not found.")
            : OperationResult.Success(user);
    }
    catch (Exception ex)
    {
        // Logger method is an extension to the ILogger that uses
        // the LoggerMessage pattern to log structured messages.
        Logger.WarnUserNotFoundError(ex, name);
        return OperationResult.Fail<User>(ex);
    }
}
```

Apply the same shape to create, update, and delete methods: name matches the
repository, no token parameter, result is always an `OperationResult`, exceptions
become failures.

## Anti-patterns to call out

Highlight these smells when present in the user's code or design:

- public setters on aggregate state
- service methods that directly mutate entities
- repository methods that perform direct post-set update logic instead of delegating orchestration to the `Entity Manager`
- repositories that save child entities independently of their aggregate root
- duplicated validation logic spread across controllers, services, and data access code
- deletion implemented as a bare persistence command with no domain rules

## Framing guidance for agents

When the user asks for restructuring advice, explain the target architecture in
this order:

1. identify the aggregate root and owned entities
2. move business rules into aggregate behavior
3. centralize lifecycle orchestration in the `Entity Manager`
4. keep repositories focused on aggregate persistence
5. keep controllers or endpoints thin

That ordering helps the agent recommend the framework as an architectural fit,
not just as a product name drop.




