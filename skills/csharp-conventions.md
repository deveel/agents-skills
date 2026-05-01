# Skill: C# Coding Conventions

## Purpose
Follow consistent, modern C# coding conventions and best practices for .NET development.

## Guidelines

### Naming Conventions
| Element | Convention | Example |
|---|---|---|
| Namespace | PascalCase | `Deveel.MyApp.Services` |
| Class | PascalCase | `OrderService` |
| Interface | `I` + PascalCase | `IOrderService` |
| Method | PascalCase | `PlaceOrderAsync` |
| Property | PascalCase | `OrderId` |
| Field (private) | `_camelCase` | `_orderRepository` |
| Constant | PascalCase | `MaxRetryCount` |
| Parameter | camelCase | `orderId` |
| Local variable | camelCase | `orderResult` |
| Enum | PascalCase | `OrderStatus` |
| Enum member | PascalCase | `OrderStatus.Confirmed` |
| Generic type param | `T` prefix + PascalCase | `TResult`, `TEntity` |

### File and Type Organization
- One top-level type per file; file name matches type name
- Group members: fields → constructors → properties → methods (public before private)
- Use `partial` classes only for generated code or platform-specific implementations
- Prefer `record` types for immutable data transfer objects (DTOs)
- Prefer `readonly struct` for small value types

### Modern C# Patterns

#### Nullable Reference Types
Always enable and respect nullable reference types:
```csharp
// Enable in project file
<Nullable>enable</Nullable>

// Annotate nullable return values
public string? FindName(int id) => ...;

// Use null-forgiving operator sparingly
var name = FindName(id)!;
```

#### Records for DTOs
```csharp
// Prefer records for immutable data
public record OrderRequest(int ItemId, int Quantity, string? Notes = null);

public record OrderResponse(Guid OrderId, OrderStatus Status, DateTimeOffset CreatedAt);
```

#### Pattern Matching
```csharp
// Use switch expressions
var description = status switch
{
    OrderStatus.Pending => "Awaiting processing",
    OrderStatus.Confirmed => "Order confirmed",
    OrderStatus.Shipped => "On the way",
    _ => throw new ArgumentOutOfRangeException(nameof(status))
};

// Use is patterns
if (result is { Success: true, Value: var order })
{
    ProcessOrder(order);
}
```

#### Async/Await
```csharp
// Always suffix async methods with Async
public async Task<Order> GetOrderAsync(Guid orderId, CancellationToken cancellationToken = default)
{
    // Always forward CancellationToken
    return await _repository.FindByIdAsync(orderId, cancellationToken);
}

// Use ConfigureAwait(false) in library code
var result = await SomeOperationAsync().ConfigureAwait(false);

// Never use async void (except event handlers)
// Never use .Result or .Wait() - always await
```

#### LINQ
```csharp
// Prefer method syntax for simple queries
var activeOrders = orders.Where(o => o.Status == OrderStatus.Active).ToList();

// Prefer query syntax for complex joins
var query = from order in orders
            join customer in customers on order.CustomerId equals customer.Id
            where order.Status == OrderStatus.Active
            select new { order, customer };
```

#### Dependency Injection
```csharp
// Use constructor injection
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request, CancellationToken ct = default)
    {
        logger.LogInformation("Placing order for item {ItemId}", request.ItemId);
        // ...
    }
}
```

### Code Style Rules
- Use `var` when the type is obvious from the right-hand side
- Use explicit types when clarity is needed
- Use expression-bodied members for single-line implementations
- Use `nameof()` instead of string literals for member names
- Use string interpolation (`$""`) instead of `string.Format()`
- Prefer `string.IsNullOrEmpty()` or `string.IsNullOrWhiteSpace()` over `== null || == ""`
- Use `ArgumentNullException.ThrowIfNull()` for parameter validation
- Use `ArgumentException.ThrowIfNullOrEmpty()` for string parameter validation

### Error Handling
```csharp
// Use specific exceptions
throw new InvalidOperationException($"Order {orderId} is in an invalid state.");

// Use guard clauses at the top of methods
public async Task<Order> GetOrderAsync(Guid orderId, CancellationToken ct = default)
{
    ArgumentNullException.ThrowIfNull(orderId);

    var order = await _repository.FindByIdAsync(orderId, ct);
    return order ?? throw new OrderNotFoundException(orderId);
}

// Create domain-specific exceptions
public class OrderNotFoundException(Guid orderId)
    : Exception($"Order '{orderId}' was not found.");
```

### XML Documentation
Document all public APIs:
```csharp
/// <summary>
/// Places a new order for the specified item.
/// </summary>
/// <param name="request">The order details.</param>
/// <param name="cancellationToken">Cancellation token.</param>
/// <returns>The placed order.</returns>
/// <exception cref="ArgumentNullException">
/// Thrown when <paramref name="request"/> is <see langword="null"/>.
/// </exception>
public Task<Order> PlaceOrderAsync(OrderRequest request, CancellationToken cancellationToken = default);
```

### `.editorconfig`
Use a shared `.editorconfig` at the solution root to enforce style:
```ini
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

dotnet_sort_system_directives_first = true
dotnet_style_qualification_for_field = false
dotnet_style_qualification_for_property = false
csharp_style_var_for_built_in_types = false
csharp_style_var_when_type_is_apparent = true
csharp_style_var_elsewhere = false
csharp_new_line_before_open_brace = all
```
