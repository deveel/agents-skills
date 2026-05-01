# GitHub Copilot Instructions – .NET Development

You are an expert .NET developer. Always follow the guidelines below when working with .NET codebases.

## Project Structure

- Use a solution (`.sln`) at the repository root for multi-project repositories.
- Place production code under `src/` and test projects under `tests/`, mirroring the source structure.
- Use `Directory.Build.props` at the solution root to share MSBuild properties:
  - `<TargetFramework>net9.0</TargetFramework>`
  - `<Nullable>enable</Nullable>`
  - `<ImplicitUsings>enable</ImplicitUsings>`
  - `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`
- Use `Directory.Packages.props` with `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` to centralise NuGet versions.

## C# Coding Conventions

- Enable and respect nullable reference types; annotate all nullable returns with `?`.
- Use `record` types for immutable DTOs and value objects.
- Use `var` only when the type is obvious from the right-hand side.
- Always suffix async methods with `Async` and accept a `CancellationToken` parameter that defaults to `default`.
- Use primary constructors (C# 12+) for simple dependency injection.
- Use `ArgumentNullException.ThrowIfNull()` and `ArgumentException.ThrowIfNullOrEmpty()` for parameter validation.
- Use `nameof()` instead of string literals for member names in exceptions and logging.
- Use pattern matching (`switch` expressions, `is` patterns) over chains of `if`/`else if`.
- Never use `.Result` or `.Wait()` on `Task`—always `await`.
- Document all public APIs with XML doc comments.

## Testing

- Use **xUnit** as the test framework with **FluentAssertions** for assertions.
- Use **NSubstitute** for mocking.
- Follow the naming convention: `MethodName_StateUnderTest_ExpectedBehavior`.
- Organise tests using AAA (Arrange, Act, Assert) with blank-line separators.
- Use `[Theory]` and `[InlineData]` for parameterised tests.
- Use `Testcontainers` for integration tests requiring a real database.

## ASP.NET Core

- Prefer **Minimal API** with `MapGroup` for new services; use controller-based API for complex existing codebases.
- Always return `IResult` or `ActionResult<T>` with proper status codes.
- Register `AddProblemDetails()` and `UseExceptionHandler()` for RFC 9457 problem responses.
- Add `AddHealthChecks()` and expose `/health` and `/health/ready` endpoints.
- Use `FluentValidation` for request validation.
- Use typed configuration (`IOptions<T>`) with `GetSection` binding.

## Entity Framework Core

- Use separate `IEntityTypeConfiguration<T>` classes applied via `ApplyConfigurationsFromAssembly`.
- Use `AsNoTracking()` for read-only queries.
- Use `AsSplitQuery()` when including multiple collections to avoid cartesian explosion.
- Run migrations with `dotnet ef migrations add` and apply them with `dotnet ef database update`.
- Never call `SaveChangesAsync` from a controller or handler; use a Unit of Work or repository pattern.

## Logging

- Use structured logging via `ILogger<T>` with message templates (not string interpolation).
- Define log messages using `[LoggerMessage]` source-generated partial methods for high-performance logging.
- Use `LogLevel.Information` for normal business events, `LogLevel.Warning` for unexpected-but-recoverable situations, and `LogLevel.Error` for failures.
- Integrate **OpenTelemetry** for traces and metrics in production services.

## Packages & Dependencies

- Never add a NuGet package without checking for known vulnerabilities (`dotnet list package --vulnerable`).
- Always use the latest stable version of packages unless a specific version is required.
- Prefer `Microsoft.Extensions.*` abstractions over direct infrastructure dependencies in domain/application layers.

## CI/CD

- Use GitHub Actions with `actions/setup-dotnet@v4` for CI pipelines.
- Always restore → build → test in sequence. Cache NuGet packages with `actions/cache@v4`.
- Collect code coverage with `--collect:"XPlat Code Coverage"` and upload to Codecov.
- Scan for vulnerable packages in every CI run.

## Docker

- Use multi-stage Dockerfiles: `sdk` image for build, `aspnet` image for runtime.
- Run the container process as a non-root user.
- Set `ASPNETCORE_URLS=http://+:8080` and expose port `8080`.
- Always add a `.dockerignore` that excludes `bin/`, `obj/`, `.git/`, and test results.
