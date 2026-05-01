# Skill: .NET Docker and Containerization

## Purpose
Containerize .NET applications using Docker following best practices for production deployments.

## Guidelines

### Dockerfile (Multi-stage build)
Use multi-stage builds to create small, secure production images:

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy project files and restore (layer cache optimization)
COPY Directory.Build.props Directory.Packages.props ./
COPY src/MyApp.Api/MyApp.Api.csproj src/MyApp.Api/
COPY src/MyApp.Domain/MyApp.Domain.csproj src/MyApp.Domain/
COPY src/MyApp.Infrastructure/MyApp.Infrastructure.csproj src/MyApp.Infrastructure/
RUN dotnet restore src/MyApp.Api/MyApp.Api.csproj

# Copy source and build
COPY src/ src/
RUN dotnet publish src/MyApp.Api/MyApp.Api.csproj \
    -c Release \
    -o /app/publish \
    --no-restore

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app

# Security: run as non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

COPY --from=build --chown=appuser:appuser /app/publish .

# Configure ASP.NET Core
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### .dockerignore
```
**/.git
**/.vs
**/bin
**/obj
**/*.user
**/TestResults
**/coverage
.env
*.md
!README.md
```

### docker-compose.yml (Development)
```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - Database__ConnectionString=Host=postgres;Database=myapp;Username=myapp;Password=secret
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Build and Run
```bash
# Build image
docker build -t myapp:latest .

# Run container
docker run -p 8080:8080 myapp:latest

# Run with docker compose
docker compose up --build

# Stop services
docker compose down
```

### Environment Variables and Configuration
ASP.NET Core supports environment variable configuration natively. Use the `__` separator for nested keys:

```bash
# Equivalent to appsettings: { "Database": { "ConnectionString": "..." } }
Database__ConnectionString=Host=...;Database=...;
```

Use secrets in production via environment variables (never hardcode in Dockerfile):
```yaml
environment:
  - Database__ConnectionString=${DATABASE_CONNECTION_STRING}
```

### Health Checks in Docker
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

Configure in `docker-compose.yml`:
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### Kubernetes (K8s) Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  labels:
    app: myapp-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp-api
  template:
    metadata:
      labels:
        app: myapp-api
    spec:
      containers:
        - name: api
          image: myapp:latest
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: Database__ConnectionString
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: database-connection-string
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Image Tagging Strategy
```bash
# Tag with version and latest
docker build -t myregistry.io/myapp:1.2.3 -t myregistry.io/myapp:latest .

# Push to registry
docker push myregistry.io/myapp:1.2.3
docker push myregistry.io/myapp:latest
```
