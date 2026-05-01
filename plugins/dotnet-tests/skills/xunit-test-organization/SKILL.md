---
name: xunit-organization
description: >-
  Guides the agent in creating, organizing, and maintaining xUnit test projects
  for .NET class libraries. Use this skill when asked to create a new test
  project, add or refactor tests, write unit or integration tests, set up
  fixtures or traits, configure code coverage, or organize test data with Bogus.
license: MIT
metadata:
  author: Antonello Provenzano
  version: "1.0"
  compatibility:
    - github-copilot
    - claude-code
    - openai-codex
---

# xUnit Test Organization

---

## 1. Project Structure

### Naming convention

| Project type | Suffix | Example | `IsTestProject` |
|---|---|---|---|
| Executable test project | `.XUnit` | `MyLib.XUnit` | `true` (default) |
| Shared test support library | `.Testing` | `MyLib.Testing` | `false` |
| Library under test | _(none)_ | `MyLib` | _(not set)_ |

The `.XUnit` suffix signals the framework in use without locking the name
permanently — if a different framework is adopted in the future, a new
`{LibraryName}.NUnit` (or similar) project can coexist alongside it.

Shared test support libraries (fixtures, builders, fakes) that are referenced
by multiple test projects but do not contain executable tests are named with
the `.Testing` suffix and have `<IsTestProject>false</IsTestProject>` in their
`.csproj`. This prevents `Directory.Build.props` from injecting xUnit runner
packages into them.

### Folder layout

```
src/
  MyLib/
    MyLib.csproj
test/
  Directory.Build.props           ← test-scope build settings
  coverlet.runsettings            ← test-scope coverage settings
  MyLib.XUnit/
    MyLib.XUnit.csproj          ← IsTestProject: true (default)
    Unit/
      MyClassTests.cs
    Integration/
      MyClassIntegrationTests.cs
    Fixtures/
      MyClassFixture.cs
      MyCollectionFixture.cs
  MyLib.Testing/
    MyLib.Testing.csproj        ← IsTestProject: false
    Builders/
      OrderBuilder.cs
    Fakes/
      FakeOrderRepository.cs
```

Rules:
- Test project folder lives under `tests/` at the solution root, mirroring `src/`
- Unit and integration tests are separated into `Unit/` and `Integration/` subfolders
- Never mix unit and integration tests in the same file
- Keep shared xUnit runner and coverage packages in `Directory.Build.props`
- Add package references directly to a test `.csproj` only when they are
  project-specific and not broadly needed by other test projects

### Directory.Build.props

Place `Directory.Build.props` in the `tests/` folder so its scope stays within
test projects only. It injects shared test dependencies conditionally based on
`<IsTestProject>` and `<TargetFramework>`, so individual `.csproj` files stay
minimal unless they need project-specific packages.

If your repository uses `tests/` (plural) instead of `test/`, apply the same
pattern in that folder.

```xml
<Project>

    <!-- ============================================================
         Defaults applied to projects under test/
    ============================================================ -->
    <PropertyGroup>
        <IsPackable>false</IsPackable>
        <!-- IsTestProject defaults to true; shared support libraries opt out explicitly -->
        <IsTestProject Condition="'$(IsTestProject)' == ''">true</IsTestProject>
    </PropertyGroup>

    <!-- ============================================================
         Packages injected into ALL projects under test/
         (both executable test projects and shared support libraries)
    ============================================================ -->
    <ItemGroup>
        <PackageReference Include="Bogus" Version="34.*" />
    </ItemGroup>

    <!-- ============================================================
         Packages injected ONLY into executable test projects
         (IsTestProject = true)
    ============================================================ -->

    <!-- .NET 8+ → xUnit v3 + Microsoft Testing Platform -->
    <ItemGroup Condition="'$(IsTestProject)' == 'true'
             and $([MSBuild]::IsTargetFrameworkCompatible('$(TargetFramework)', 'net8.0'))">
        <PackageReference Include="xunit.v3" Version="1.*" />
        <PackageReference Include="xunit.runner.visualstudio" Version="3.*" />
        <PackageReference Include="Microsoft.Testing.Extensions.CodeCoverage" Version="17.*" />
        <PackageReference Include="coverlet.collector" Version="6.*">
            <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
            <PrivateAssets>all</PrivateAssets>
        </PackageReference>
    </ItemGroup>

    <!-- .NET 6/7 → xUnit v2 + VSTest -->
    <ItemGroup Condition="'$(IsTestProject)' == 'true'
             and !$([MSBuild]::IsTargetFrameworkCompatible('$(TargetFramework)', 'net8.0'))">
        <PackageReference Include="xunit" Version="2.*" />
        <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
        <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
        <PackageReference Include="coverlet.collector" Version="6.*">
            <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
            <PrivateAssets>all</PrivateAssets>
        </PackageReference>
    </ItemGroup>

    <!-- MTP properties for .NET 8+ test projects -->
    <PropertyGroup Condition="'$(IsTestProject)' == 'true'
                 and $([MSBuild]::IsTargetFrameworkCompatible('$(TargetFramework)', 'net8.0'))">
        <TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>
        <TestingPlatformCaptureOutput>false</TestingPlatformCaptureOutput>
    </PropertyGroup>

</Project>
```

### Minimal .csproj for an executable test project

Because `Directory.Build.props` handles shared packages and MTP properties,
the `.csproj` typically only needs to declare the target framework and project
reference:

```xml
<!-- tests/MyLib.XUnit/MyLib.XUnit.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net9.0</TargetFramework>
    </PropertyGroup>

    <ItemGroup>
        <ProjectReference Include="..\..\src\MyLib\MyLib.csproj" />
    </ItemGroup>
</Project>
```

If a specific test project requires additional packages that are not shared by
other test projects, add those `PackageReference` entries in that project's
`.csproj`.

### Minimal .csproj for a shared test support library

```xml
<!-- tests/MyLib.Testing/MyLib.Testing.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net9.0</TargetFramework>
        <!-- Opt out of xUnit runner injection from Directory.Build.props -->
        <IsTestProject>false</IsTestProject>
    </PropertyGroup>

    <ItemGroup>
        <ProjectReference Include="..\..\src\MyLib\MyLib.csproj" />
    </ItemGroup>
</Project>
```

The `.XUnit` project then references the `.Testing` library:

```xml
<ItemGroup>
    <ProjectReference Include="..\..\src\MyLib\MyLib.csproj" />
    <ProjectReference Include="..\MyLib.Testing\MyLib.Testing.csproj" />
</ItemGroup>
```

### Multi-targeting

For solutions that target multiple frameworks, set `<TargetFrameworks>` in
the `.csproj` — `Directory.Build.props` conditions handle the rest:

```xml
<TargetFrameworks>net6.0;net8.0;net9.0</TargetFrameworks>
```

No further changes to `Directory.Build.props` are needed.

### Code coverage — running and reporting

Shared package setup is handled by `test/Directory.Build.props` (see above).
Use the following commands to collect and report coverage.

**MTP (.NET 8+):**
```bash
# Collect in Cobertura format
dotnet test --coverage --coverage-output ./coverage --coverage-output-format cobertura

# Filter and collect together
dotnet test --coverage -- --filter "Category=Unit"
```

**VSTest (.NET 6/7):**
```bash
dotnet test --collect:"XPlat Code Coverage" \
            --settings test/coverlet.runsettings \
            --results-directory ./coverage
```

### Coverage configuration — coverlet.runsettings

Place a `coverlet.runsettings` file in the `test/` folder for VSTest exclusions.
MTP picks up the same exclusion patterns via MSBuild properties in
`Directory.Build.props` or the `--coverage-include` / `--coverage-exclude` flags.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<RunSettings>
    <DataCollectionRunSettings>
        <DataCollectors>
            <DataCollector friendlyName="XPlat Code Coverage">
                <Configuration>
                    <Format>cobertura</Format>
                    <Exclude>[*.XUnit]*,[*.Testing]*,[*.Migrations]*</Exclude>
                    <ExcludeByAttribute>GeneratedCodeAttribute,ExcludeFromCodeCoverageAttribute</ExcludeByAttribute>
                    <ExcludeByFile>**/Program.cs,**/Migrations/**</ExcludeByFile>
                    <SingleHit>false</SingleHit>
                    <IncludeTestAssembly>false</IncludeTestAssembly>
                </Configuration>
            </DataCollector>
        </DataCollectors>
    </DataCollectionRunSettings>
</RunSettings>
```

> **Note:** FluentAssertions is **not used** in this project. All assertions
> use xUnit's built-in `Assert` class exclusively.
---

## 2. Test Naming Convention

All test methods follow the pattern:

```
Should_{ExpectedResult}_When_{Scenario}
```

Examples:
```csharp
Should_ReturnNull_When_InputIsEmpty()
Should_ThrowArgumentException_When_ValueIsNegative()
Should_ParseCorrectly_When_ValidJsonIsProvided()
Should_ReturnCachedResult_When_CalledTwice()
```

Rules:
- Use PascalCase for each segment
- `ExpectedResult` describes the observable outcome
- `Scenario` describes the condition or input state
- Never abbreviate — clarity is more important than brevity
- Test class name: `{ClassName}Tests` (e.g. `OrderServiceTests`) — note the
  class suffix remains `Tests` even though the project suffix is `.XUnit`

---

## 3. Test Class Structure

Each test class must follow this internal layout order:

```csharp
namespace MyLib.XUnit.Unit;

public class OrderServiceTests : IClassFixture<OrderServiceFixture>
{
    // 1. Fields
    private readonly OrderServiceFixture _fixture;

    // 2. Constructor
    public OrderServiceTests(OrderServiceFixture fixture)
    {
        _fixture = fixture;
    }

    // 3. [Fact] tests grouped by the method under test,
    //    with a comment header per group
    #region ProcessOrder

    [Fact]
    public void Should_ReturnConfirmed_When_OrderIsValid()
    {
        // Arrange
        var order = _fixture.BuildValidOrder();

        // Act
        var result = _fixture.Sut.ProcessOrder(order);

        // Assert
        Assert.Equal(OrderStatus.Confirmed, result.Status);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void Should_ThrowArgumentException_When_QuantityIsNotPositive(int quantity)
    {
        // Arrange
        var order = _fixture.BuildOrderWithQuantity(quantity);

        // Act & Assert
        var ex = Assert.Throws<ArgumentException>(
            () => _fixture.Sut.ProcessOrder(order));
        Assert.Contains("quantity", ex.Message);
    }

    #endregion
}
```

For xUnit v3 async tests that call async APIs with a `CancellationToken`
parameter, always flow the test runner token into those calls:

```csharp
[Fact]
public async Task Should_ReturnOrder_When_OrderExistsAsync()
{
    // Arrange
    var cancellationToken = TestContext.Current.CancellationToken;
    var order = _fixture.BuildValidOrder();
    await _fixture.Sut.SaveAsync(order, cancellationToken);

    // Act
    var result = await _fixture.Sut.GetByIdAsync(order.Id, cancellationToken);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(order.Id, result.Id);
}
```

Rules:
- Always use `// Arrange`, `// Act`, `// Assert` comments — no exceptions
- One assertion concept per `[Fact]`; multiple related assertions are allowed
  within the same concept (e.g. asserting multiple properties of the same result)
- `[Theory]` is required whenever the same logic is tested with multiple inputs;
  never duplicate `[Fact]` methods with hardcoded variations
- Use `[InlineData]` for simple scalar values; use `[MemberData]` pointing to a
  `public static IEnumerable<object[]>` property for complex objects
- In xUnit v3 async `[Fact]` / `[Theory]` methods, when the async API under test
  accepts a `CancellationToken`, pass `TestContext.Current.CancellationToken`
  through every awaited call so cancelled test runs do not remain blocked waiting
  on the underlying operation

---

## 4. Test Data — Bogus & Randomization

All test data must be generated using **Bogus**. Never use hardcoded literals
for names, emails, addresses, IDs, amounts, or any domain value that could
realistically vary. Randomized data catches edge cases that fixed values miss
and prevents tests from accidentally passing due to magic constants.

### Required package
```xml
<PackageReference Include="Bogus" Version="34.*" />
```

### Faker placement

Fakers are defined as `static readonly` fields inside the fixture that owns
the entity, not inside individual test methods. This keeps data generation
centralized and reusable.

```csharp
// test/MyLib.Testing/Fixtures/OrderServiceFixture.cs
namespace MyLib.Testing.Fixtures;

public class OrderServiceFixture
{
    // One Faker<T> per domain entity, defined once
    private static readonly Faker<Order> OrderFaker = new Faker<Order>()
        .RuleFor(o => o.Id, f => f.Random.Guid())
        .RuleFor(o => o.ProductId, f => f.Commerce.Ean13())
        .RuleFor(o => o.Quantity, f => f.Random.Int(1, 100))
        .RuleFor(o => o.CustomerName, f => f.Name.FullName())
        .RuleFor(o => o.Email, f => f.Internet.Email())
        .RuleFor(o => o.CreatedAt, f => f.Date.RecentOffset().UtcDateTime);

    public OrderService Sut { get; }

    public OrderServiceFixture()
    {
        var repo = new InMemoryOrderRepository();
        Sut = new OrderService(repo);
    }

    // Builder methods delegate to the Faker, with overrides for specific scenarios
    public Order BuildValidOrder() =>
        OrderFaker.Generate();

    public Order BuildOrderWithQuantity(int quantity) =>
        OrderFaker.Clone().RuleFor(o => o.Quantity, quantity).Generate();

    public IEnumerable<Order> BuildOrders(int count) =>
        OrderFaker.Generate(count);
}
```

### Seeding for reproducibility

When a test fails due to randomized data, the seed must be reproducible.
Use a fixed seed only in `[Theory]` / `[MemberData]` scenarios where
determinism is required; elsewhere let Bogus randomize freely.

```csharp
// Deterministic seed for MemberData — use when the exact values matter
private static readonly Faker<Order> SeededOrderFaker =
    new Faker<Order>("en").UseSeed(12345)
        .RuleFor(o => o.Id, f => f.Random.Guid())
        .RuleFor(o => o.Quantity, f => f.Random.Int(1, 100));
```

When a randomly seeded test fails, xUnit's output includes the data values —
capture them and promote the failing case to a named `[InlineData]` or
`[MemberData]` entry so it becomes a permanent regression test.

### MemberData with Bogus

Use `[MemberData]` with a Bogus-generated dataset for `[Theory]` tests on
complex objects:

```csharp
public static IEnumerable<object[]> InvalidOrders =>
    new Faker<Order>()
        .RuleFor(o => o.Id, f => f.Random.Guid())
        .RuleFor(o => o.Quantity, f => f.Random.Int(-100, 0)) // always invalid
        .RuleFor(o => o.ProductId, f => f.Commerce.Ean13())
        .Generate(5)
        .Select(o => new object[] { o });

[Theory]
[MemberData(nameof(InvalidOrders))]
[Trait("Category", "Unit")]
[Trait("Layer", "Domain")]
[Trait("Feature", "OrderProcessing")]
public void Should_ThrowArgumentException_When_QuantityIsNotPositive(Order order)
{
    // Act & Assert
    var ex = Assert.Throws<ArgumentException>(
        () => _fixture.Sut.ProcessOrder(order));
    Assert.Contains("quantity", ex.Message);
}
```

### Rules
- Always use `Faker<T>` with explicit `RuleFor` for every property — never
  rely on Bogus auto-generation without rules, as it can produce unexpected nulls
- Define one `Faker<T>` per entity type per fixture — do not instantiate
  new `Faker<T>` inside individual test methods
- Use `f.Random.Guid()` instead of `Guid.NewGuid()` so randomization flows
  through the Bogus seed
- Use locale `"en"` explicitly when string format matters (e.g. phone numbers,
  postcodes): `new Faker<Order>("en")`
- Prefer exact-value assertions when the expected value is deterministic and
  meaningful for the test intent; when using non-deterministic randomized data,
  assert on behaviour, shape, or range instead (e.g. `Assert.True(result > 0)`)

---

## 5. Fixtures

Fixtures that are reused across multiple test projects belong in the shared
`.Testing` project (e.g. `MyLib.Testing`). Fixtures used by a single test
project can live in its `Fixtures/` subfolder directly.

### ClassFixture — shared across all tests in one class
Use when the system under test (SUT) is expensive to construct and is stateless
between tests, or when its state is always reset in the fixture constructor.

```csharp
// test/MyLib.Testing/Fixtures/OrderServiceFixture.cs
namespace MyLib.Testing.Fixtures;

public class OrderServiceFixture
{
    // Bogus Faker defined once — see Section 4 for full rules
    private static readonly Faker<Order> OrderFaker = new Faker<Order>("en")
        .RuleFor(o => o.Id, f => f.Random.Guid())
        .RuleFor(o => o.ProductId, f => f.Commerce.Ean13())
        .RuleFor(o => o.Quantity, f => f.Random.Int(1, 100))
        .RuleFor(o => o.CustomerName, f => f.Name.FullName())
        .RuleFor(o => o.Email, f => f.Internet.Email());

    public OrderService Sut { get; }

    public OrderServiceFixture()
    {
        var repo = new InMemoryOrderRepository();
        Sut = new OrderService(repo);
    }

    public Order BuildValidOrder() =>
        OrderFaker.Generate();

    public Order BuildOrderWithQuantity(int quantity) =>
        OrderFaker.Clone().RuleFor(o => o.Quantity, quantity).Generate();
}
```

### CollectionFixture — shared across multiple test classes
Use for expensive shared infrastructure (e.g. a single in-memory database or
HTTP server) that must be shared across more than one test class.

```csharp
// test/MyLib.Testing/Fixtures/DatabaseFixture.cs
namespace MyLib.Testing.Fixtures;

public class DatabaseFixture : IAsyncLifetime
{
    public IDbConnection Connection { get; private set; } = default!;

    public async Task InitializeAsync()
    {
        Connection = new SqliteConnection("Data Source=:memory:");
        await Connection.OpenAsync();
        // Run migrations / seed data
    }

    public async Task DisposeAsync() => await Connection.DisposeAsync();
}

[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }
```

Apply to test classes with:
```csharp
[Collection("Database")]
public class OrderRepositoryTests
{
    private readonly DatabaseFixture _db;
    public OrderRepositoryTests(DatabaseFixture db) => _db = db;
    // ...
}
```

---

## 6. Traits — Categorization

All tests must be decorated with `[Trait]` to allow selective test runs in CI.

### Standard trait keys

| Key        | Values                              |
|------------|-------------------------------------|
| `Category` | `Unit`, `Integration`               |
| `Layer`    | `Domain`, `Application`, `Infrastructure` |
| `Feature`  | The feature or module name (e.g. `OrderProcessing`) |

Examples:
```csharp
[Fact]
[Trait("Category", "Unit")]
[Trait("Layer", "Domain")]
[Trait("Feature", "OrderProcessing")]
public void Should_ReturnConfirmed_When_OrderIsValid() { ... }

[Fact]
[Trait("Category", "Integration")]
[Trait("Layer", "Infrastructure")]
[Trait("Feature", "OrderProcessing")]
public void Should_PersistOrder_When_ProcessingSucceeds() { ... }
```

To run only unit tests in CI:
```bash
dotnet test --filter "Category=Unit"
dotnet test --filter "Category=Integration"
dotnet test --filter "Feature=OrderProcessing"
```

---

## 7. Integration Tests

Integration tests live in the `Integration/` subfolder and follow the same
naming and structure conventions as unit tests, with these additional rules:

- Class must be decorated with `[Collection("...")]` referencing a
  `CollectionFixture` that owns the shared infrastructure
- Tests must be fully isolated — always clean up or reset state in the fixture's
  `InitializeAsync` / `DisposeAsync`
- No mocking of infrastructure in integration tests; use real implementations
  or in-memory equivalents (e.g. `SqliteConnection`, `WebApplicationFactory`)
- Use `IAsyncLifetime` on fixtures that manage async resources

```csharp
[Collection("Database")]
[Trait("Category", "Integration")]
[Trait("Layer", "Infrastructure")]
[Trait("Feature", "OrderRepository")]
public class OrderRepositoryIntegrationTests
{
    private readonly DatabaseFixture _db;

    public OrderRepositoryIntegrationTests(DatabaseFixture db) => _db = db;

    [Fact]
    public async Task Should_PersistOrder_When_SaveAsyncIsCalled()
    {
        // Arrange
        var cancellationToken = TestContext.Current.CancellationToken;
        var repo = new OrderRepository(_db.Connection);
        var order = _db.OrderFaker.Generate(); // use Bogus, not hardcoded values

        // Act
        await repo.SaveAsync(order, cancellationToken);
        var retrieved = await repo.GetByIdAsync(order.Id, cancellationToken);

        // Assert
        Assert.NotNull(retrieved);
        Assert.Equal(order.Quantity, retrieved.Quantity);
    }
}
```

---

## 8. What the Agent Must Never Do

- Do not place tests directly in the solution root or `src/` folder
- Do not name test projects with the `.Tests` suffix — use `.XUnit` for executable test projects
- Do not duplicate shared test package references in individual test `.csproj`
  files — keep common dependencies in `Directory.Build.props` and add local
  `PackageReference` entries only for project-specific needs
- Do not set `<IsTestProject>false</IsTestProject>` on executable test projects —
  only shared support libraries (`.Testing`) use this opt-out
- Do not use `[Fact]` where `[Theory]` is more appropriate
- Do not write test methods without all three AAA comment sections
- Do not use FluentAssertions — always use xUnit's built-in `Assert` class
- Do not add `Microsoft.NET.Test.Sdk` to projects targeting .NET 8+ with xUnit v3 — it conflicts with MTP
- Do not add `xunit` (v2) packages alongside `xunit.v3` — they are mutually exclusive
- Do not share mutable state between tests without a fixture
- Do not name test methods with generic names like `Test1`, `TestProcess`, etc.
- Do not skip adding `[Trait]` attributes
- Do not hardcode test data values (names, IDs, quantities, emails, etc.) — always use Bogus
- Do not instantiate `Faker<T>` inside individual test methods — define it in the fixture
- Do not use brittle exact-value assertions against non-deterministic
  Bogus-generated values; use exact assertions when values are intentionally
  deterministic (for example seeded or explicitly overridden)
- In xUnit v3 async tests, do not omit `TestContext.Current.CancellationToken`
  when calling async APIs that already expose a `CancellationToken` parameter

---

## 9. Local References

Additional supporting material for this skill is available in the
[`references/README.md`](./references/README.md) index beside this file.

- [`references/README.md`](./references/README.md) — overview and topic index
- [`references/xunit.md`](./references/xunit.md) — xUnit framework, fixtures, theories, and traits
- [`references/dotnet-testing-platform.md`](./references/dotnet-testing-platform.md) — runner and platform guidance by target framework
- [`references/coverage.md`](./references/coverage.md) — coverage collection and reporting references
- [`references/bogus.md`](./references/bogus.md) — randomized test data guidance
- [`references/msbuild.md`](./references/msbuild.md) — tests-folder-scoped `Directory.Build.props` guidance for test project configuration

Use these files when deeper background or authoritative external links are
helpful, while treating this `SKILL.md` as the primary instruction source.

