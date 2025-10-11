# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

Conductor is an open-source microservices orchestration engine originally built at Netflix, now maintained by Orkes and the open-source community. It enables developers to create complex, resilient workflows that orchestrate interactions between services, databases, and external systems.

**Key Technologies:**
- **Backend**: Java 17, Spring Boot 3.3.5, Gradle
- **Frontend**: React 18, Node.js 14+, Yarn
- **Databases**: Redis (default), PostgreSQL, MySQL, Cassandra
- **Search/Indexing**: Elasticsearch 7.x, OpenSearch 2.x
- **Messaging**: Kafka, NATS, AMQP, AWS SQS
- **Storage**: AWS S3, Azure Blob Storage, PostgreSQL

## Common Development Commands

### Server Development

```bash
# Build the entire project
./gradlew build

# Run the server with default configuration (in-memory persistence)
cd server && ../gradlew bootRun

# Run with specific configuration (e.g., Redis + Elasticsearch)
cd server && CONFIG_PROP=config-redis.properties ../gradlew bootRun

# Run tests
./gradlew test

# Run tests for a specific module
./gradlew :conductor-core:test

# Format code (Spotless)
./gradlew spotlessApply

# Check code formatting
./gradlew spotlessCheck

# Build server JAR
./gradlew :conductor-server:bootJar
```

### UI Development

```bash
# Install dependencies
cd ui && yarn install

# Start development server (requires server running on port 8080)
cd ui && yarn start

# Build for production
cd ui && yarn build

# Run tests
cd ui && yarn test

# Format code
cd ui && yarn prettier

# Serve production build locally
cd ui && yarn serve-build
```

### Docker Development

```bash
# Start complete stack (server + Redis + Elasticsearch)
docker compose -f docker/docker-compose.yaml up

# Start with PostgreSQL backend
docker compose -f docker/docker-compose-postgres.yaml up

# Start with MySQL backend
docker compose -f docker/docker-compose-mysql.yaml up

# Build custom server image
docker build -f docker/server/Dockerfile -t conductor:server .
```

## Architecture Overview

### Module Structure

Conductor is organized as a multi-module Gradle project with clear separation of concerns:

**Core Modules:**
- `conductor-core`: Core workflow engine, task management, and execution logic
- `conductor-common`: Shared utilities, models, and constants
- `conductor-server`: Spring Boot application and REST API endpoints
- `conductor-rest`: REST client and API definitions

**Persistence Modules:**
- `conductor-redis-persistence`: Redis-based persistence (default)
- `conductor-postgres-persistence`: PostgreSQL persistence
- `conductor-mysql-persistence`: MySQL persistence
- `conductor-cassandra-persistence`: Cassandra persistence

**Search/Indexing Modules:**
- `conductor-es7-persistence`: Elasticsearch 7.x integration
- `conductor-os-persistence`: OpenSearch integration

**Event Queue Modules:**
- `conductor-kafka-event-queue`: Kafka integration
- `conductor-awssqs-event-queue`: AWS SQS integration
- `conductor-amqp`: AMQP/RabbitMQ integration

**Task Types:**
- `conductor-http-task`: HTTP task execution
- `conductor-json-jq-task`: JSON transformation tasks

**Client SDKs:**
- `conductor-grpc-client`: gRPC client for high-performance communication
- `conductor-annotations`: Annotations for workflow definitions

### Key Architectural Concepts

**Workflow Engine**: The core orchestration engine manages workflow state, task scheduling, and execution. Workflows are defined as JSON and can be managed independently of services.

**Task Workers**: External services that implement business logic and register as task workers. Workers poll for tasks and execute them asynchronously.

**Persistence Layer**: Pluggable persistence supporting multiple backends. Default Redis setup provides fast querying, while SQL databases offer durability.

**Event System**: Asynchronous event processing for workflow state changes, enabling real-time monitoring and external integrations.

**UI Architecture**: React SPA with Material-UI components, providing workflow visualization, monitoring, and management capabilities.

## Development Workflow

### Running Locally for Development

1. **Server Only** (in-memory, no persistence):
   ```bash
   cd server && ../gradlew bootRun
   ```
   Access API at http://localhost:8080/swagger-ui/index.html

2. **Full Stack with Docker**:
   ```bash
   docker compose -f docker/docker-compose.yaml up
   ```
   - Server: http://localhost:8080
   - UI: http://localhost:8127
   - Redis: http://localhost:7379
   - Elasticsearch: http://localhost:9201

3. **UI Development** (requires server running):
   ```bash
   cd ui && yarn install && yarn start
   ```
   Access UI at http://localhost:5000

### Testing

- **Unit Tests**: Each module has comprehensive unit tests
- **Integration Tests**: Test harness module provides integration test utilities
- **UI Tests**: Cypress end-to-end tests for UI workflows

### Configuration

Configuration is handled through Spring Boot properties files:
- `docker/server/config/config-redis.properties`: Redis + Elasticsearch
- `docker/server/config/config-postgres.properties`: PostgreSQL
- `docker/server/config/config-mysql.properties`: MySQL

Environment variable `CONDUCTOR_CONFIG_FILE` or system property can override default configuration.

## Project-Specific Guidelines

### Java Code Standards
- Java 17 language features and APIs
- Google Java Format (AOSP variant) enforced by Spotless
- Lombok for reducing boilerplate (annotations processed at compile time)
- Spring Boot conventions for dependency injection and configuration

### Database Considerations
- Default Redis setup is suitable for development but not production persistence
- Use PostgreSQL or MySQL for production deployments requiring durability
- Elasticsearch/OpenSearch required for search functionality in UI

### Module Dependencies
- Core modules (`conductor-core`, `conductor-common`) should remain database-agnostic
- Persistence modules implement interfaces defined in core
- Server module wires together all components based on configuration

### Performance Notes
- Workflow execution is asynchronous and event-driven
- Task polling workers should be scaled based on load
- Database connection pooling and indexing strategies are critical for production

### Docker Development
- All necessary Docker compositions are in `/docker` directory
- Server builds include both application and UI assets
- Use specific configurations for different backend combinations

## External Dependencies

- Spring Boot 3.3.5 (framework and dependency management)
- Jackson 2.18.0 (JSON processing, version-locked across modules)
- gRPC 1.73.0 (high-performance client communication)
- Elasticsearch/OpenSearch client libraries (version-specific modules)
- Database drivers (PostgreSQL 42.7.2, MySQL, Cassandra 3.10.2)