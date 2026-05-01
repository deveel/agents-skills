# Skill: NuGet Package Management

## Purpose
Manage NuGet package dependencies in .NET projects effectively and securely.

## Guidelines

### Central Package Management
Use `Directory.Packages.props` at the solution root to centralise package version management:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- Core -->
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Logging.Abstractions" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Configuration" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Http" Version="9.0.0" />

    <!-- Testing -->
    <PackageVersion Include="xunit" Version="2.9.3" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
    <PackageVersion Include="FluentAssertions" Version="7.0.0" />
    <PackageVersion Include="NSubstitute" Version="5.3.0" />
    <PackageVersion Include="coverlet.collector" Version="6.0.3" />
  </ItemGroup>
</Project>
```

In individual project files, omit the `Version` attribute:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.DependencyInjection" />
</ItemGroup>
```

### CLI Commands
```bash
# Add a package to a project
dotnet add package Serilog

# Add a specific version
dotnet add package Serilog --version 4.2.0

# Remove a package
dotnet remove package Serilog

# List installed packages
dotnet list package

# List outdated packages
dotnet list package --outdated

# List packages with known vulnerabilities
dotnet list package --vulnerable

# Restore packages
dotnet restore

# Update all packages (requires dotnet-outdated tool)
dotnet tool install -g dotnet-outdated-tool
dotnet outdated --upgrade
```

### NuGet Configuration (`nuget.config`)
Place at the solution root for custom feeds:
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
  </packageSourceMapping>
</configuration>
```

### Recommended Packages for .NET Development

#### Logging
| Package | Purpose |
|---|---|
| `Serilog` | Structured logging |
| `Serilog.AspNetCore` | Serilog integration for ASP.NET Core |
| `Serilog.Sinks.Console` | Console sink |
| `Serilog.Sinks.File` | File sink |
| `OpenTelemetry.Extensions.Hosting` | OpenTelemetry integration |

#### Serialization
| Package | Purpose |
|---|---|
| `System.Text.Json` | Built-in JSON serialization |
| `Newtonsoft.Json` | Advanced JSON (when System.Text.Json is insufficient) |

#### HTTP and API
| Package | Purpose |
|---|---|
| `Refit` | Type-safe REST client |
| `Polly` | Resilience and transient-fault-handling |
| `Microsoft.Extensions.Http.Resilience` | Polly integration for HttpClient |

#### Database
| Package | Purpose |
|---|---|
| `Microsoft.EntityFrameworkCore` | ORM |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | PostgreSQL provider |
| `Microsoft.EntityFrameworkCore.SqlServer` | SQL Server provider |
| `Dapper` | Lightweight ORM |

#### Validation
| Package | Purpose |
|---|---|
| `FluentValidation` | Fluent validation rules |
| `FluentValidation.AspNetCore` | ASP.NET Core integration |

#### Testing
| Package | Purpose |
|---|---|
| `xunit` | Test framework |
| `FluentAssertions` | Expressive assertions |
| `NSubstitute` | Mocking framework |
| `Bogus` | Fake data generation |
| `Microsoft.AspNetCore.Mvc.Testing` | Integration testing for ASP.NET Core |
| `Testcontainers` | Docker containers in tests |

### Security
- Run `dotnet list package --vulnerable` regularly and in CI to detect vulnerable dependencies
- Use `dotnet nuget audit` (available in .NET 8+) to audit packages
- Pin package versions in production applications
- Use Source Link to enable debugging into NuGet packages
- Never include pre-release packages in production builds

### Publishing a NuGet Package
```bash
# Pack the project
dotnet pack -c Release -o ./artifacts

# Push to NuGet.org
dotnet nuget push ./artifacts/*.nupkg --api-key $NUGET_API_KEY --source https://api.nuget.org/v3/index.json
```

Use these properties in the `.csproj` for proper package metadata:
```xml
<PropertyGroup>
  <PackageId>Deveel.MyPackage</PackageId>
  <Version>1.0.0</Version>
  <Authors>Deveel</Authors>
  <Description>Description of the package</Description>
  <PackageLicenseExpression>Apache-2.0</PackageLicenseExpression>
  <PackageProjectUrl>https://github.com/deveel/my-package</PackageProjectUrl>
  <RepositoryUrl>https://github.com/deveel/my-package</RepositoryUrl>
  <PackageTags>dotnet;deveel</PackageTags>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
</PropertyGroup>
```
