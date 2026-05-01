# .NET Development Skills

This directory contains skill definitions for AI coding agents to assist with .NET development tasks.

## Available Skills

| Skill | Description |
|---|---|
| [dotnet-build.md](./dotnet-build.md) | Building .NET projects with the `dotnet` CLI, MSBuild properties, and central package management |
| [dotnet-testing.md](./dotnet-testing.md) | Writing and running tests with xUnit, FluentAssertions, NSubstitute, and code coverage |
| [csharp-conventions.md](./csharp-conventions.md) | Modern C# coding conventions, naming, patterns (records, async/await, LINQ) |
| [nuget-packages.md](./nuget-packages.md) | Managing NuGet packages, central package management, recommended packages |
| [aspnetcore-api.md](./aspnetcore-api.md) | Building ASP.NET Core Web APIs (minimal APIs and controllers) |
| [entity-framework.md](./entity-framework.md) | Entity Framework Core data access patterns and migrations |
| [dotnet-docker.md](./dotnet-docker.md) | Containerizing .NET applications with Docker and Kubernetes |
| [dotnet-cicd.md](./dotnet-cicd.md) | CI/CD pipelines for .NET with GitHub Actions |
| [dotnet-architecture.md](./dotnet-architecture.md) | Clean Architecture, DDD, CQRS, and Vertical Slice patterns |
| [dotnet-logging.md](./dotnet-logging.md) | Structured logging, OpenTelemetry, and observability |

## Usage

These skills are referenced by agent-specific configuration files in the repository root. Each agent has its own format for consuming skills:

- **GitHub Copilot**: `.github/copilot-instructions.md`
- **Cursor**: `.cursor/rules/*.mdc`
- **Claude**: `CLAUDE.md`
- **Cline**: `.clinerules`
- **Windsurf**: `.windsurfrules`
