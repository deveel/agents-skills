# Skill: .NET Testing

## Purpose
Write and run automated tests for .NET applications using the `dotnet test` CLI and common .NET testing frameworks.

## Guidelines

### Preferred Testing Frameworks
- **Unit tests**: [xUnit](https://xunit.net/) (preferred) or NUnit
- **Assertion library**: [FluentAssertions](https://fluentassertions.com/) for expressive assertions
- **Mocking**: [NSubstitute](https://nsubstitute.github.io/) (preferred) or Moq
- **Integration/API tests**: Microsoft.AspNetCore.Mvc.Testing + xUnit
- **Database integration tests**: [Respawn](https://github.com/jbogard/Respawn) for database cleanup, EF Core `UseInMemoryDatabase` for unit tests

### Test Project Setup
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="xunit" />
    <PackageReference Include="xunit.runner.visualstudio" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="FluentAssertions" />
    <PackageReference Include="NSubstitute" />
    <PackageReference Include="coverlet.collector" />
  </ItemGroup>
</Project>
```

### Test Naming Convention
Follow the pattern: `MethodName_StateUnderTest_ExpectedBehavior`

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrder_WhenItemIsAvailable_ReturnsConfirmedOrder()
    {
        // Arrange
        var repository = Substitute.For<IOrderRepository>();
        var service = new OrderService(repository);

        // Act
        var result = await service.PlaceOrderAsync(new OrderRequest { ItemId = 1, Quantity = 2 });

        // Assert
        result.Should().NotBeNull();
        result.Status.Should().Be(OrderStatus.Confirmed);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public async Task PlaceOrder_WhenQuantityIsInvalid_ThrowsArgumentException(int quantity)
    {
        // Arrange
        var service = new OrderService(Substitute.For<IOrderRepository>());

        // Act
        var act = async () => await service.PlaceOrderAsync(new OrderRequest { Quantity = quantity });

        // Assert
        await act.Should().ThrowAsync<ArgumentException>()
            .WithMessage("*quantity*");
    }
}
```

### Running Tests
```bash
# Run all tests
dotnet test

# Run with detailed output
dotnet test -v normal

# Run tests in a specific project
dotnet test tests/MyProject.Tests/MyProject.Tests.csproj

# Run with code coverage
dotnet test --collect:"XPlat Code Coverage"

# Run specific test by name filter
dotnet test --filter "FullyQualifiedName~OrderServiceTests"

# Run and produce TRX report
dotnet test --logger trx --results-directory TestResults
```

### Code Coverage
```bash
# Collect coverage and generate report with ReportGenerator
dotnet test --collect:"XPlat Code Coverage" -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:**/coverage.opencover.xml -targetdir:coverage -reporttypes:Html
```

### Test Organization
- Put tests in a `tests/` folder at the solution root
- Mirror the `src/` structure: `src/MyApp.Domain/` → `tests/MyApp.Domain.Tests/`
- Use `[Collection]` attribute for shared fixtures in xUnit
- Use `IClassFixture<T>` for test class-level setup/teardown

### Integration Testing with ASP.NET Core
```csharp
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrdersApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetOrders_ReturnsOkWithOrders()
    {
        var response = await _client.GetAsync("/api/orders");
        response.Should().Be200Ok();
    }
}
```
