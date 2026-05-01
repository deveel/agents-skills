# Skill: .NET Logging and Observability

## Purpose
Implement structured logging, distributed tracing, and metrics in .NET applications following OpenTelemetry standards.

## Guidelines

### Structured Logging with Serilog
```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
```

```csharp
// Program.cs
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithThreadId()
    .WriteTo.Console(new JsonFormatter())
    .CreateLogger();

builder.Host.UseSerilog();
```

### Logging Best Practices

#### Use high-performance logging with source generators
```csharp
public partial class OrderService(ILogger<OrderService> logger)
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Placing order {OrderId} for customer {CustomerId}")]
    private partial void LogPlacingOrder(Guid orderId, Guid customerId);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Order {OrderId} already exists")]
    private partial void LogOrderAlreadyExists(Guid orderId);

    [LoggerMessage(Level = LogLevel.Error, Message = "Failed to place order {OrderId}")]
    private partial void LogOrderFailed(Guid orderId, Exception exception);
}
```

#### Log levels
| Level | Use when |
|---|---|
| `Trace` | Very detailed debugging (disabled in production) |
| `Debug` | Detailed debugging information |
| `Information` | Normal application flow (requests, important business events) |
| `Warning` | Unexpected but recoverable situations |
| `Error` | Errors that prevent a specific operation from completing |
| `Critical` | System-level failures requiring immediate attention |

#### Structured logging - always use message templates, never string interpolation
```csharp
// CORRECT - structured properties for log aggregation
logger.LogInformation("Order {OrderId} placed for customer {CustomerId}", orderId, customerId);

// WRONG - string interpolation loses structure
logger.LogInformation($"Order {orderId} placed for customer {customerId}");
```

#### Scoped logging context
```csharp
using (logger.BeginScope(new Dictionary<string, object>
{
    ["CorrelationId"] = correlationId,
    ["CustomerId"] = customerId
}))
{
    // All log entries within this scope include CorrelationId and CustomerId
    logger.LogInformation("Processing order");
}
```

### OpenTelemetry
```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: "MyApp.Api",
            serviceVersion: typeof(Program).Assembly.GetName().Version?.ToString()))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddOtlpExporter(otlp =>
        {
            otlp.Endpoint = new Uri(builder.Configuration["OpenTelemetry:Endpoint"]!);
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter());
```

### Custom Metrics
```csharp
// Define metrics using a meter
public class OrderMetrics
{
    private readonly Counter<int> _ordersPlaced;
    private readonly Histogram<double> _orderProcessingDuration;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyApp.Orders");
        _ordersPlaced = meter.CreateCounter<int>("orders.placed", description: "Number of orders placed");
        _orderProcessingDuration = meter.CreateHistogram<double>(
            "orders.processing_duration",
            unit: "ms",
            description: "Time to process an order");
    }

    public void RecordOrderPlaced(string status) 
        => _ordersPlaced.Add(1, new TagList { { "status", status } });

    public void RecordProcessingDuration(double milliseconds)
        => _orderProcessingDuration.Record(milliseconds);
}

// Register and use
builder.Services.AddSingleton<OrderMetrics>();
```

### Request Logging Middleware
Configure Serilog request logging to exclude noisy paths:
```csharp
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    options.GetLevel = (httpContext, elapsed, ex) => ex != null
        ? LogEventLevel.Error
        : httpContext.Response.StatusCode > 499
            ? LogEventLevel.Error
            : LogEventLevel.Information;
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent);
    };
});
```

### Health Checks with Diagnostics
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(tags: ["ready"])
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```
