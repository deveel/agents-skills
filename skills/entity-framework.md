# Skill: Entity Framework Core

## Purpose
Use Entity Framework Core effectively for data access in .NET applications.

## Guidelines

### Setup
```bash
# Install EF Core tools globally
dotnet tool install -g dotnet-ef

# Add packages to project
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL  # or SqlServer, SQLite, etc.
dotnet add package Microsoft.EntityFrameworkCore.Design   # for migrations
```

### DbContext
```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all configurations from the assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### Entity Configuration (Fluent API)
Prefer separate configuration classes over data annotations:
```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");
        builder.HasKey(o => o.Id);

        builder.Property(o => o.Id)
            .HasColumnName("id")
            .IsRequired();

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasColumnName("status")
            .IsRequired();

        builder.Property(o => o.CreatedAt)
            .HasColumnName("created_at")
            .IsRequired();

        builder.HasOne(o => o.Customer)
            .WithMany(c => c.Orders)
            .HasForeignKey(o => o.CustomerId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.HasIndex(o => o.CustomerId);
    }
}
```

### Registering the DbContext
```csharp
// PostgreSQL
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        npgsqlOptions => npgsqlOptions.EnableRetryOnFailure(3)));

// SQL Server
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions => sqlOptions.EnableRetryOnFailure(3)));
```

### Migrations
```bash
# Add a migration
dotnet ef migrations add InitialCreate --project src/MyApp.Infrastructure --startup-project src/MyApp.Api

# Update the database
dotnet ef database update --project src/MyApp.Infrastructure --startup-project src/MyApp.Api

# Generate SQL script (for production deployments)
dotnet ef migrations script --idempotent -o migrations.sql

# Remove the last migration
dotnet ef migrations remove
```

Apply migrations at startup for development/testing:
```csharp
// In Program.cs (development only)
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await dbContext.Database.MigrateAsync();
}
```

### Repository Pattern
```csharp
// Interface
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> FindByCustomerAsync(Guid customerId, CancellationToken ct = default);
    Task<Order> AddAsync(Order order, CancellationToken ct = default);
    Task UpdateAsync(Order order, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}

// Implementation
public class OrderRepository(AppDbContext context) : IOrderRepository
{
    public async Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default)
        => await context.Orders.FindAsync([id], ct);

    public async Task<IReadOnlyList<Order>> FindByCustomerAsync(Guid customerId, CancellationToken ct = default)
        => await context.Orders
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync(ct);

    public async Task<Order> AddAsync(Order order, CancellationToken ct = default)
    {
        context.Orders.Add(order);
        await context.SaveChangesAsync(ct);
        return order;
    }

    public async Task UpdateAsync(Order order, CancellationToken ct = default)
    {
        context.Orders.Update(order);
        await context.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(Guid id, CancellationToken ct = default)
    {
        var order = await FindByIdAsync(id, ct)
            ?? throw new OrderNotFoundException(id);
        context.Orders.Remove(order);
        await context.SaveChangesAsync(ct);
    }
}
```

### Unit of Work
```csharp
public interface IUnitOfWork
{
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class UnitOfWork(AppDbContext context, IOrderRepository orders) : IUnitOfWork
{
    public IOrderRepository Orders { get; } = orders;

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => context.SaveChangesAsync(ct);
}
```

### Querying Best Practices
```csharp
// Use AsNoTracking for read-only queries
var orders = await context.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Active)
    .ToListAsync(ct);

// Use AsSplitQuery for queries with multiple Include chains (avoids cartesian explosion)
var customers = await context.Customers
    .AsSplitQuery()
    .Include(c => c.Orders)
    .Include(c => c.Addresses)
    .ToListAsync(ct);

// Use projection to avoid over-fetching
var orderSummaries = await context.Orders
    .AsNoTracking()
    .Select(o => new OrderSummary(o.Id, o.Status, o.CreatedAt))
    .ToListAsync(ct);

// Use FirstOrDefaultAsync instead of SingleOrDefaultAsync when you expect one result (better performance)
var order = await context.Orders
    .FirstOrDefaultAsync(o => o.Id == orderId, ct);
```

### Integration Testing with EF Core
```csharp
// Use a real database for integration tests (with Testcontainers)
public class OrderRepositoryTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder().Build();
    private AppDbContext _context = null!;

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_postgres.GetConnectionString())
            .Options;
        _context = new AppDbContext(options);
        await _context.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _context.DisposeAsync();
        await _postgres.DisposeAsync();
    }

    [Fact]
    public async Task FindByIdAsync_WhenOrderExists_ReturnsOrder()
    {
        var repository = new OrderRepository(_context);
        var order = new Order { Id = Guid.NewGuid(), Status = OrderStatus.Pending };
        await repository.AddAsync(order);

        var found = await repository.FindByIdAsync(order.Id);

        found.Should().NotBeNull();
        found!.Id.Should().Be(order.Id);
    }
}
```
