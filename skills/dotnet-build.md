# Skill: .NET Build

## Purpose
Build .NET projects and solutions using the `dotnet` CLI.

## Guidelines

### Project Structure
- Use solution files (`.sln`) at the root of multi-project repositories
- Organize projects under a `src/` directory and tests under `tests/`
- Use `Directory.Build.props` at solution root to share common MSBuild properties
- Use `Directory.Packages.props` with central package management to keep NuGet versions consistent

### Build Commands
```bash
# Restore dependencies
dotnet restore

# Build in Debug mode (default)
dotnet build

# Build in Release mode
dotnet build -c Release

# Build a specific project
dotnet build src/MyProject/MyProject.csproj

# Build and suppress output
dotnet build -v quiet
```

### MSBuild Properties
Always set these properties in `Directory.Build.props` or individual project files:
```xml
<PropertyGroup>
  <TargetFramework>net9.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <AnalysisMode>All</AnalysisMode>
</PropertyGroup>
```

### Common `Directory.Build.props` Template
```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
</Project>
```

### Central Package Management (`Directory.Packages.props`)
```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- Define package versions here -->
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="9.0.0" />
  </ItemGroup>
</Project>
```

### Error Handling
- Always address all compiler warnings; treat warnings as errors in CI
- Use `.editorconfig` and `.globalconfig` to enforce code style rules
- Run `dotnet format` to auto-fix formatting issues before committing
