# NetCorePal Cloud Framework
NetCorePal Cloud Framework is a tactical Domain-Driven Design (DDD) framework built on ASP.NET Core and .NET 9.0. It provides extensions for context passing, distributed transactions, multi-tenancy, multi-environment deployment, and comprehensive DDD patterns.

Always reference these instructions first and fallback to search or bash commands only when you encounter unexpected information that does not match the info here.

## Working Effectively

### Prerequisites and Setup
- Install .NET 9.0 SDK (specific version required):
  ```bash
  wget -O dotnet-install.sh https://dotnet.microsoft.com/download/dotnet/scripts/v1/dotnet-install.sh
  chmod +x dotnet-install.sh
  ./dotnet-install.sh --version 9.0.100-rc.1.24452.12
  export PATH="$HOME/.dotnet:$PATH"
  ```
- Verify installation: `dotnet --version` should show `9.0.100-rc.1.24452.12`

### Bootstrap, Build, and Test Commands
- **Restore dependencies**: `dotnet restore` -- takes 50 seconds. NEVER CANCEL. Set timeout to 90+ seconds.
- **Build all projects**: `dotnet build -c Release --no-restore` -- takes 41 seconds. NEVER CANCEL. Set timeout to 60+ seconds.
- **Build single project**: `dotnet build test/NetCorePal.Web/NetCorePal.Web.csproj --no-restore` -- takes 18 seconds for complex projects.
- **Run all unit tests**: `dotnet test --framework net9.0` -- basic unit tests take 2-3 seconds per project. Integration tests need external dependencies.
- **Run specific project tests**: `dotnet test test/NetCorePal.Extensions.Domain.Abstractions.UnitTests/NetCorePal.Extensions.Domain.Abstractions.UnitTests.csproj --framework net9.0` -- takes 2-3 seconds.

### External Dependencies for Integration Tests
The framework requires external services for full integration testing. Use Docker to set up:
```bash
# Redis (required for distributed locking and caching)
docker run -p 6379:6379 -d redis:7.0

# RabbitMQ (required for CAP distributed events)
docker run -p 5672:5672 -p 15672:15672 -d rabbitmq:3.9-management

# PostgreSQL (for .NET 9.0 tests)
docker run -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=test@123 -d postgres:latest

# MySQL (for .NET 8.0 tests)
docker run -p 3306:3306 -e MYSQL_ROOT_PASSWORD=test@123 -d mysql:8.0
```

**Alternative**: Use the provided docker-compose for complete environment:
```bash
cd docker && docker-compose up -d
```

## Validation

### Always Run These Validation Steps
1. **Build validation**: Always run `dotnet build -c Release --no-restore` after making changes
2. **Unit test validation**: Run framework-specific unit tests: `dotnet test --framework net9.0` for projects that don't require external dependencies
3. **Integration test validation**: Only run integration tests if you have set up the required external services
4. **SDK verification**: Run `dotnet --version` to ensure .NET 9.0 SDK (9.0.100-rc.1.24452.12) is installed

### Manual Testing Scenarios
- **Test the sample web application**: The `test/NetCorePal.Web` project demonstrates framework usage
  - Build: `dotnet build test/NetCorePal.Web/NetCorePal.Web.csproj --no-restore` (takes ~18 seconds)
  - Run: `dotnet run --project test/NetCorePal.Web/NetCorePal.Web.csproj` (requires external dependencies: Redis, RabbitMQ, PostgreSQL)
  - **Without external dependencies**: Application will build successfully but fail at runtime with connection errors
  - **With external dependencies**: Application starts on default port with health check at `/health` and root endpoint at `/`

### Sample Project Configuration
```
NetCorePal.Web demonstrates:
- Domain entities and events
- Repository pattern usage
- Distributed transactions
- Context passing
- Multi-environment setup
- Health checks and monitoring
```

### Troubleshooting Common Issues
- **Build failures**: Usually due to missing .NET 9.0 SDK - verify version with `dotnet --version` (should show 9.0.100-rc.1.24452.12)
- **Test failures with Redis/RabbitMQ errors**: External dependencies not available - either set up Docker services or run only unit tests with `--framework net9.0`
- **Multi-target framework errors**: Project targets both net8.0 and net9.0. Use `--framework net9.0` to run only .NET 9.0 tests if .NET 8.0 runtime missing
- **Security warnings in restore**: Package vulnerability warnings (NU1903) are expected and can be ignored for development
- **Source generator warnings**: CS9113 and CS1591 warnings are common and expected in sample projects

### Critical Timing Guidelines
- **NEVER CANCEL**: Build takes 45+ seconds, restore takes 50+ seconds. Use timeouts of 60+ minutes for build commands and 30+ minutes for test commands.
- **Package restore**: First run takes longest due to NuGet downloads
- **Source generators**: Build includes source generation which adds time
- **Multi-targeting**: Projects target both .NET 8.0 and 9.0, doubling compilation time

## Project Structure and Key Areas

### Core Framework Projects (src/)
- **AspNetCore**: ASP.NET Core extensions and middleware
- **Domain.Abstractions**: Core DDD building blocks (Entity, ValueObject, AggregateRoot, DomainEvent)
- **Domain.SourceGenerators**: Source generators for strongly-typed IDs
- **Repository.EntityFrameworkCore**: Repository pattern implementation
- **DistributedTransactions.CAP**: Distributed transaction support using CAP
- **Context.AspNetCore**: Context passing for HTTP requests
- **Context.CAP**: Context passing for message handling
- **Primitives**: Core primitives and exception handling
- **MultiEnv**: Multi-environment support for gray deployments

### Test Projects (test/)
- **NetCorePal.Web**: Sample web application demonstrating framework usage
- **NetCorePal.Web.UnitTests**: Unit tests for the sample application
- **NetCorePal.Extensions.Domain.Abstractions.UnitTests**: Core domain unit tests
- **NetCorePal.Extensions.*.UnitTests**: Framework component unit tests

### Build Configuration
- **Directory.Build.props**: Global MSBuild properties and package versions
- **Directory.Build.targets**: Version-specific package references
- **global.json**: .NET SDK version pinning
- **.github/workflows/**: CI/CD pipelines (dotnet.yml for preview builds)

## Working with Specific Areas

### Domain-Driven Design Development
- **Entities**: Extend `Entity<TId>` or `Entity` from Domain.Abstractions
- **Value Objects**: Implement `ValueObject` base class
- **Domain Events**: Implement `IDomainEvent` and use `DomainEventPublisher`
- **Strongly-typed IDs**: Use source generators from Domain.SourceGenerators

### Repository Pattern
- **Repository interfaces**: Define in Repository.Abstractions
- **Entity Framework implementation**: Use Repository.EntityFrameworkCore
- **Unit of Work**: Integrated with repository pattern

### Distributed Transactions
- **CAP integration**: Use DistributedTransactions.CAP for event-driven architectures
- **Database providers**: MySQL, PostgreSQL, SqlServer support available
- **Message brokers**: RabbitMQ integration included

### Context Passing
- **HTTP requests**: Use Context.AspNetCore for request-scoped data
- **Message handling**: Use Context.CAP for message-scoped data
- **Service calls**: Use Context.Shared for cross-service context

## Common Tasks

The following are outputs from frequently run commands. Reference them instead of viewing, searching, or running bash commands to save time.

### Repository Structure
```
/src/                   # Framework source code
  AspNetCore/          # ASP.NET Core extensions
  Domain.Abstractions/ # DDD core abstractions
  Repository.*/        # Repository pattern implementations
  DistributedTransactions.*/ # Distributed transaction support
  Context.*/           # Context passing implementations
  Primitives/          # Core primitives
  MultiEnv/           # Multi-environment support
/test/                 # Test projects
  NetCorePal.Web/     # Sample web application
  *.UnitTests/        # Unit test projects
/docs/                 # Documentation
/docker/              # Docker compose files
/.github/workflows/   # CI/CD pipelines
```

### Package Information
```
Target Frameworks: net8.0, net9.0
Key Dependencies:
- Microsoft.AspNetCore.*
- Microsoft.EntityFrameworkCore.*
- DotNetCore.CAP.*
- MediatR
- StackExchange.Redis
- FluentValidation
- Serilog
```

### Example Command Outputs

#### dotnet --version
```
9.0.100-rc.1.24452.12
```

#### dotnet restore timing
```
Build succeeded with 55 warning(s) in 49.5s
```

#### dotnet build timing  
```
Build succeeded with 124 warning(s) in 41.3s
```

#### dotnet test simple unit test
```
Test summary: total: 24, failed: 0, succeeded: 24, skipped: 0, duration: 0.9s
Build succeeded in 1.5s
```

## Development Guidelines

### Making Changes
1. **Always build first**: `dotnet build -c Release --no-restore` to ensure current state
2. **Make targeted changes**: This is a framework - changes should be minimal and focused
3. **Test your changes**: Run relevant unit tests after modifications
4. **Validate integration**: If changing core framework components, test with NetCorePal.Web
5. **Check warnings**: Security warnings are expected, but new build warnings should be addressed

### Framework Extension Points
- **Custom repositories**: Extend base repository classes
- **Custom domain events**: Implement IDomainEvent
- **Custom context processors**: Implement IContextProcessor
- **Custom validation**: Use FluentValidation integration
- **Custom source generators**: Extend existing generators in SourceGenerators projects

### Debugging and Diagnostics
- **Logging**: Uses Serilog throughout
- **Health checks**: Available at `/health` endpoint
- **CAP dashboard**: Available when CAP services are running
- **OpenTelemetry**: Integrated for distributed tracing
- **SkyWalking**: Alternative APM integration available