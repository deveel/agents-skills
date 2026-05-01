# .NET Development Guidelines for Claude

You are an expert .NET developer. Follow these guidelines for all .NET development tasks in this repository.

## Repository Structure

Use Clean Architecture with this layout:
```
src/
  MyApp.Domain/         # Entities, value objects, domain events, interfaces
  MyApp.Application/    # Commands, queries, handlers, validators, DTOs
  MyApp.Infrastructure/ # EF Core, repositories, external services
  MyApp.Api/            # ASP.NET Core entry point, DI configuration
tests/
  MyApp.Domain.Tests/
  MyApp.Application.Tests/
  MyApp.Infrastructure.Tests/
  MyApp.Api.Tests/
```

## Build Configuration

Every solution must have:
- `Directory.Build.props` with `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, `<TargetFramework>net9.0</TargetFramework>`
- `Directory.Packages.props` with `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>`

## C# Coding Style

- Use `record` types for DTOs and value objects
- Use primary constructors for dependency injection
- Use switch expressions and pattern matching over if/else chains
- Always suffix async methods with `Async` and accept `CancellationToken cancellationToken = default`
- Enable nullable reference types; annotate all nullable members
- Use `ArgumentNullException.ThrowIfNull()` for null guard checks
- Use structured logging with `ILogger<T>` message templates (not string interpolation)
- Use `[LoggerMessage]` partial methods for high-frequency log calls

## Testing

Preferred stack: **xUnit** + **FluentAssertions** + **NSubstitute** + **Testcontainers**

- Name tests: `MethodName_StateUnderTest_ExpectedBehavior`
- Structure tests with Arrange / Act / Assert sections separated by blank lines
- Use `[Theory]` + `[InlineData]` for parameterised tests
- Use `Testcontainers` for integration tests that require a real database or external service

## ASP.NET Core

- Use Minimal API with `MapGroup` for new microservices
- Register `AddProblemDetails()` + `UseExceptionHandler()` for RFC 9457 error responses
- Add health checks: `/health/live` (liveness) and `/health/ready` (readiness)
- Use `FluentValidation` for request validation
- Use `IOptions<T>` for typed configuration

## Entity Framework Core

- Use separate `IEntityTypeConfiguration<T>` classes, applied via `ApplyConfigurationsFromAssembly`
- Use `AsNoTracking()` for read-only queries
- Use the repository pattern; never call `SaveChangesAsync` from controllers
- Run migrations: `dotnet ef migrations add <Name>` and `dotnet ef database update`

## Docker

- Use multi-stage Dockerfiles: `sdk` to build, `aspnet` to run
- Run as a non-root user in the runtime stage
- Expose port `8080` and set `ASPNETCORE_URLS=http://+:8080`

## CI/CD (GitHub Actions)

Workflow order: restore → build → test → pack/publish

Key actions:
- `actions/setup-dotnet@v4`
- `actions/cache@v4` for NuGet packages
- Collect code coverage: `--collect:"XPlat Code Coverage"`
- Scan for vulnerable packages: `dotnet list package --vulnerable`

## Commands Reference

```bash
dotnet restore                        # Restore packages
dotnet build -c Release               # Build release
dotnet test --collect:"XPlat Code Coverage"  # Test with coverage
dotnet ef migrations add <Name>       # Add EF migration
dotnet ef database update             # Apply migrations
dotnet pack -c Release -o artifacts   # Create NuGet package
dotnet format                         # Auto-fix formatting
dotnet list package --vulnerable      # Check for security issues
```
