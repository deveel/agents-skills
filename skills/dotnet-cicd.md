# Skill: .NET CI/CD with GitHub Actions

## Purpose
Set up robust CI/CD pipelines for .NET projects using GitHub Actions.

## Guidelines

### Continuous Integration Workflow
```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  DOTNET_VERSION: '9.0.x'
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: true
  DOTNET_CLI_TELEMETRY_OPTOUT: true
  NUGET_PACKAGES: ${{ github.workspace }}/.nuget/packages

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ${{ env.NUGET_PACKAGES }}
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj', '**/Directory.Packages.props') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -c Release

      - name: Run tests
        run: |
          dotnet test --no-build -c Release \
            --logger trx \
            --results-directory TestResults \
            --collect:"XPlat Code Coverage" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: TestResults/**/*.trx

      - name: Upload coverage reports
        uses: codecov/codecov-action@v4
        with:
          files: '**/coverage.opencover.xml'
          fail_ci_if_error: false
```

### NuGet Publish Workflow
```yaml
# .github/workflows/publish.yml
name: Publish

on:
  release:
    types: [published]

env:
  DOTNET_VERSION: '9.0.x'
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: true
  DOTNET_CLI_TELEMETRY_OPTOUT: true

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build -c Release --no-restore

      - name: Test
        run: dotnet test -c Release --no-build

      - name: Pack
        run: dotnet pack -c Release --no-build -o artifacts -p:PackageVersion=${{ github.event.release.tag_name }}

      - name: Push to NuGet.org
        run: |
          dotnet nuget push artifacts/*.nupkg \
            --api-key ${{ secrets.NUGET_API_KEY }} \
            --source https://api.nuget.org/v3/index.json \
            --skip-duplicate
```

### Docker Build and Push Workflow
```yaml
# .github/workflows/docker.yml
name: Docker

on:
  push:
    branches: [main]
  release:
    types: [published]

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
```

### Security Scanning Workflow
```yaml
# .github/workflows/security.yml
name: Security

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1' # Weekly on Mondays

jobs:
  vulnerability-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Check for vulnerable packages
        run: dotnet list package --vulnerable --include-transitive

  codeql:
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Build
        run: dotnet build

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```

### Release Versioning
Use [MinVer](https://github.com/adamralph/minver) or [Nerdbank.GitVersioning](https://github.com/dotnet/Nerdbank.GitVersioning) for automatic semantic versioning from Git tags:

```bash
# MinVer
dotnet add package MinVer

# Create a release tag
git tag v1.2.3
git push origin v1.2.3
```

```xml
<!-- In Directory.Build.props -->
<PropertyGroup>
  <MinVerDefaultPreReleaseIdentifiers>preview.0</MinVerDefaultPreReleaseIdentifiers>
</PropertyGroup>
```
