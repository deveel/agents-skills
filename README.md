# agents-skills

A collection of skills and configuration files for AI coding agents, dedicated to .NET development.

This repository provides ready-to-use instructions and rules for multiple AI coding agents — including GitHub Copilot, Claude, Cursor, Cline, and Windsurf — covering the most common .NET development tasks.

## Supported Agents

| Agent | Configuration file(s) | Format |
|---|---|---|
| **GitHub Copilot** | `.github/copilot-instructions.md` | Markdown custom instructions |
| **GitHub Copilot (Cloud Agent)** | `.github/copilot-setup-steps.yml` | YAML setup steps |
| **Cursor** | `.cursor/rules/*.mdc` | MDC rules (preferred) |
| **Cursor (legacy)** | `.cursorrules` | Plain text rules |
| **Claude** | `CLAUDE.md` | Markdown project instructions |
| **Cline** | `.clinerules` | Plain text rules |
| **Windsurf** | `.windsurfrules` | Plain text rules |

## Skills

The [`skills/`](./skills/) directory contains detailed skill definitions for common .NET development tasks:

| Skill | Description |
|---|---|
| [dotnet-build.md](./skills/dotnet-build.md) | `dotnet` CLI, MSBuild properties, `Directory.Build.props`, central package management |
| [dotnet-testing.md](./skills/dotnet-testing.md) | xUnit, FluentAssertions, NSubstitute, code coverage, integration testing |
| [csharp-conventions.md](./skills/csharp-conventions.md) | Naming, nullable reference types, records, async/await, LINQ, XML docs |
| [nuget-packages.md](./skills/nuget-packages.md) | Package management, `Directory.Packages.props`, recommended packages, security |
| [aspnetcore-api.md](./skills/aspnetcore-api.md) | Minimal API, controllers, ProblemDetails, FluentValidation, health checks |
| [entity-framework.md](./skills/entity-framework.md) | DbContext, entity configuration, migrations, repository pattern, querying |
| [dotnet-docker.md](./skills/dotnet-docker.md) | Multi-stage Dockerfile, docker-compose, Kubernetes deployment |
| [dotnet-cicd.md](./skills/dotnet-cicd.md) | GitHub Actions workflows for CI, NuGet publish, Docker build, security scanning |
| [dotnet-architecture.md](./skills/dotnet-architecture.md) | Clean Architecture, DDD, CQRS with MediatR, domain events, Result pattern |
| [dotnet-logging.md](./skills/dotnet-logging.md) | Serilog, structured logging, OpenTelemetry, custom metrics, health diagnostics |

## How to Use

### GitHub Copilot

The `.github/copilot-instructions.md` file is automatically picked up by GitHub Copilot in VS Code and on GitHub.com when working in this repository. Copy it to your own repository's `.github/` directory to apply the same .NET guidelines.

For the **GitHub Copilot Cloud Agent**, `.github/copilot-setup-steps.yml` pre-installs .NET global tools (`dotnet-ef`, `dotnet-outdated-tool`, `dotnet-reportgenerator-globaltool`, `csharpier`).

### Cursor

The `.cursor/rules/` directory contains MDC rule files that are automatically applied by Cursor based on glob patterns:

- `csharp-conventions.mdc` — applied to all `*.cs` files
- `dotnet-projects.mdc` — applied to `*.csproj`, `*.sln`, and MSBuild files
- `dotnet-testing.mdc` — applied to test files
- `aspnetcore-api.mdc` — applied to ASP.NET Core files
- `entity-framework.mdc` — applied to EF Core files

The legacy `.cursorrules` file provides a single-file summary for older Cursor versions.

### Claude

`CLAUDE.md` in the repository root is automatically read by Claude as project context when you start a conversation in this directory.

### Cline

`.clinerules` is automatically picked up by Cline as project-level rules.

### Windsurf

`.windsurfrules` is automatically picked up by Windsurf (Cascade) as project-level rules.

## Contributing

To add or update a skill:

1. Create or edit the relevant file in the `skills/` directory.
2. Update the agent-specific configuration files to reflect the new or changed guidance.
3. Open a pull request with a clear description of the skill being added or modified.

## License

This project is licensed under the [Apache License 2.0](LICENSE).
