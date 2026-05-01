# Skill: .NET Architecture Patterns

## Purpose
Apply well-established architecture patterns for .NET applications, including Clean Architecture, Domain-Driven Design (DDD), and CQRS.

## Guidelines

### Clean Architecture
Organize the solution into layers with clear dependency rules (dependencies point inward):

```
Solution/
├── src/
│   ├── MyApp.Domain/          # Core: entities, value objects, domain events, interfaces
│   ├── MyApp.Application/     # Use cases: commands, queries, handlers, DTOs, validators
│   ├── MyApp.Infrastructure/  # EF Core, repositories, external services, messaging
│   └── MyApp.Api/             # ASP.NET Core entry point: controllers/endpoints, DI setup
└── tests/
    ├── MyApp.Domain.Tests/
    ├── MyApp.Application.Tests/
    ├── MyApp.Infrastructure.Tests/
    └── MyApp.Api.Tests/
```

**Dependency rule**: `Domain` ← `Application` ← `Infrastructure` and `Api`

### Domain Layer
Contains business logic with no infrastructure dependencies:

```csharp
// Entity (has identity)
public class Order
{
    private readonly List<OrderLine> _lines = [];

    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
    public DateTimeOffset CreatedAt { get; private set; } = DateTimeOffset.UtcNow;

    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    private Order() { } // EF Core constructor

    public static Order Create(Guid customerId)
    {
        var order = new Order { CustomerId = customerId };
        order._domainEvents.Add(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }

    public void AddLine(int itemId, int quantity, decimal unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Can only add lines to draft orders.");

        _lines.Add(new OrderLine(itemId, quantity, unitPrice));
    }

    public void Submit()
    {
        if (!_lines.Any())
            throw new InvalidOperationException("Cannot submit an order with no lines.");
        Status = OrderStatus.Submitted;
        _domainEvents.Add(new OrderSubmittedEvent(Id));
    }
}

// Value Object (no identity, equality by value)
public record Money(decimal Amount, string Currency)
{
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Cannot add amounts in different currencies.");
        return new Money(a.Amount + b.Amount, a.Currency);
    }
}
```

### CQRS with MediatR
Separate read (queries) and write (commands) operations:

```bash
dotnet add package MediatR
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
```

```csharp
// Register MediatR
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<Application.AssemblyReference>());

// Command
public record PlaceOrderCommand(Guid CustomerId, IEnumerable<OrderLineItem> Items) 
    : IRequest<OrderResponse>;

// Command Handler
public class PlaceOrderCommandHandler(IOrderRepository repository, IUnitOfWork unitOfWork)
    : IRequestHandler<PlaceOrderCommand, OrderResponse>
{
    public async Task<OrderResponse> Handle(PlaceOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId);
        foreach (var item in request.Items)
            order.AddLine(item.ItemId, item.Quantity, item.UnitPrice);
        order.Submit();

        await repository.AddAsync(order, ct);
        await unitOfWork.SaveChangesAsync(ct);

        return new OrderResponse(order.Id, order.Status, order.CreatedAt);
    }
}

// Query
public record GetOrderQuery(Guid OrderId) : IRequest<OrderResponse?>;

// Query Handler
public class GetOrderQueryHandler(IOrderReadRepository readRepository)
    : IRequestHandler<GetOrderQuery, OrderResponse?>
{
    public async Task<OrderResponse?> Handle(GetOrderQuery request, CancellationToken ct)
        => await readRepository.FindByIdAsync(request.OrderId, ct);
}
```

### Domain Events
Publish and handle domain events using MediatR:

```csharp
// Domain event
public record OrderSubmittedEvent(Guid OrderId) : INotification;

// Event handler
public class OrderSubmittedEventHandler(IEmailService emailService)
    : INotificationHandler<OrderSubmittedEvent>
{
    public async Task Handle(OrderSubmittedEvent notification, CancellationToken ct)
    {
        await emailService.SendOrderConfirmationAsync(notification.OrderId, ct);
    }
}

// Publish events after saving (using a SaveChanges interceptor)
public class DomainEventInterceptor(IPublisher publisher) : SaveChangesInterceptor
{
    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        var entries = eventData.Context!.ChangeTracker.Entries<Entity>()
            .SelectMany(e => e.Entity.DomainEvents)
            .ToList();

        foreach (var domainEvent in entries)
            await publisher.Publish(domainEvent, ct);

        return result;
    }
}
```

### Vertical Slice Architecture (Alternative)
Organize by feature instead of layer:

```
Features/
├── Orders/
│   ├── CreateOrder/
│   │   ├── CreateOrderCommand.cs
│   │   ├── CreateOrderHandler.cs
│   │   ├── CreateOrderValidator.cs
│   │   └── CreateOrderEndpoint.cs
│   ├── GetOrder/
│   │   ├── GetOrderQuery.cs
│   │   ├── GetOrderHandler.cs
│   │   └── GetOrderEndpoint.cs
│   └── ListOrders/
│       ├── ListOrdersQuery.cs
│       ├── ListOrdersHandler.cs
│       └── ListOrdersEndpoint.cs
└── Customers/
    └── ...
```

### Result Pattern
Avoid exceptions for expected business failures:

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(T value) { IsSuccess = true; Value = value; }
    private Result(string error) { IsSuccess = false; Error = error; }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error) => new(error);
}

// Usage
public async Task<Result<Order>> PlaceOrderAsync(PlaceOrderCommand command)
{
    if (!command.Items.Any())
        return Result<Order>.Failure("Order must have at least one item.");

    // ...
    return Result<Order>.Success(order);
}
```
