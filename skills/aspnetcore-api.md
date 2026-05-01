# Skill: ASP.NET Core API Development

## Purpose
Build well-structured, production-ready ASP.NET Core Web APIs following current best practices.

## Guidelines

### Project Setup
Use the minimal API or controller-based approach depending on complexity:

```bash
# Minimal API (recommended for simple/microservice scenarios)
dotnet new webapi -n MyApi --use-minimal-apis

# Controller-based API
dotnet new webapi -n MyApi
```

### Program.cs Structure (Minimal API)
```csharp
var builder = WebApplication.CreateBuilder(args);

// Register services
builder.Services.AddOpenApi();
builder.Services.AddProblemDetails();

// Application-specific services
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();

var app = builder.Build();

// Configure middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();
app.UseExceptionHandler();

// Map endpoints
app.MapOrderEndpoints();

app.Run();
```

### Endpoint Organization (Minimal API)
Group related endpoints using extension methods:
```csharp
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/api/orders")
            .WithTags("Orders")
            .WithOpenApi();

        group.MapGet("/", GetOrdersAsync)
            .Produces<IEnumerable<OrderResponse>>();

        group.MapGet("/{id:guid}", GetOrderByIdAsync)
            .Produces<OrderResponse>()
            .ProducesProblem(StatusCodes.Status404NotFound);

        group.MapPost("/", CreateOrderAsync)
            .Produces<OrderResponse>(StatusCodes.Status201Created)
            .ProducesValidationProblem();

        group.MapDelete("/{id:guid}", DeleteOrderAsync)
            .Produces(StatusCodes.Status204NoContent)
            .ProducesProblem(StatusCodes.Status404NotFound);

        return routes;
    }

    private static async Task<IResult> GetOrdersAsync(
        IOrderService service,
        CancellationToken ct)
    {
        var orders = await service.GetOrdersAsync(ct);
        return Results.Ok(orders);
    }

    private static async Task<IResult> GetOrderByIdAsync(
        Guid id,
        IOrderService service,
        CancellationToken ct)
    {
        var order = await service.GetOrderAsync(id, ct);
        return order is null ? Results.Problem(statusCode: 404) : Results.Ok(order);
    }

    private static async Task<IResult> CreateOrderAsync(
        OrderRequest request,
        IOrderService service,
        CancellationToken ct)
    {
        var order = await service.CreateOrderAsync(request, ct);
        return Results.Created($"/api/orders/{order.Id}", order);
    }

    private static async Task<IResult> DeleteOrderAsync(
        Guid id,
        IOrderService service,
        CancellationToken ct)
    {
        await service.DeleteOrderAsync(id, ct);
        return Results.NoContent();
    }
}
```

### Controller-Based API
```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class OrdersController(IOrderService service, ILogger<OrdersController> logger) : ControllerBase
{
    [HttpGet]
    [ProducesResponseType<IEnumerable<OrderResponse>>(StatusCodes.Status200OK)]
    public async Task<IActionResult> GetOrders(CancellationToken cancellationToken)
    {
        var orders = await service.GetOrdersAsync(cancellationToken);
        return Ok(orders);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken cancellationToken)
    {
        var order = await service.GetOrderAsync(id, cancellationToken);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateOrder(
        [FromBody] OrderRequest request,
        CancellationToken cancellationToken)
    {
        var order = await service.CreateOrderAsync(request, cancellationToken);
        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
    }
}
```

### Error Handling with ProblemDetails
```csharp
// In Program.cs
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Instance = $"{context.HttpContext.Request.Method} {context.HttpContext.Request.Path}";
    };
});

// Use exception handler middleware
app.UseExceptionHandler();

// Map specific exceptions to status codes
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();

// Custom exception handler
public class NotFoundExceptionHandler(IProblemDetailsService problemDetailsService) 
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not NotFoundException notFoundEx)
            return false;

        httpContext.Response.StatusCode = StatusCodes.Status404NotFound;
        await problemDetailsService.WriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = { Title = "Not Found", Detail = notFoundEx.Message }
        });

        return true;
    }
}
```

### Validation with FluentValidation
```csharp
// Register validators
builder.Services.AddValidatorsFromAssemblyContaining<Program>();
builder.Services.AddFluentValidationAutoValidation();

// Define validator
public class OrderRequestValidator : AbstractValidator<OrderRequest>
{
    public OrderRequestValidator()
    {
        RuleFor(x => x.ItemId).GreaterThan(0);
        RuleFor(x => x.Quantity).InclusiveBetween(1, 100);
        RuleFor(x => x.Notes).MaximumLength(500).When(x => x.Notes is not null);
    }
}
```

### Authentication and Authorization
```csharp
// JWT Bearer authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
    });

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("AdminOnly", policy => policy.RequireRole("admin"));

// Apply to endpoints
group.RequireAuthorization();
group.RequireAuthorization("AdminOnly");
```

### Configuration
```csharp
// Typed configuration
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection(DatabaseOptions.SectionName));

public class DatabaseOptions
{
    public const string SectionName = "Database";
    public string ConnectionString { get; set; } = string.Empty;
    public int MaxRetryCount { get; set; } = 3;
}

// appsettings.json
{
  "Database": {
    "ConnectionString": "Server=...;Database=...;",
    "MaxRetryCount": 3
  }
}
```

### Health Checks
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddCheck<ExternalApiHealthCheck>("external-api");

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```
