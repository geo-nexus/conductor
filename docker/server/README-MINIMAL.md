# Conductor OSS - Minimal Docker Image

This is a minimal Conductor OSS Docker image designed for production deployments with only essential components.

## Components Included

### ✅ Essential Components
- **Core Module** - Workflow orchestration engine
- **REST API** - HTTP endpoints for workflow management
- **gRPC Server** - High-performance gRPC protocol support
- **PostgreSQL Persistence** - Database storage with distributed locking
- **Kafka Event Queue** - Event-driven workflow capabilities
- **System Tasks**:
  - HTTP Task - Make HTTP/REST calls
  - JSON-JQ Task - JSON transformation
  - Kafka Task - Kafka integration
- **Metrics** - Prometheus-compatible observability
- **Workflow Event Listener** - Status change notifications and archival

### ❌ Excluded Components
- **UI** - React web interface (reduces image size by ~300MB)
- **Nginx** - Web server for UI
- **Elasticsearch/OpenSearch** - Search indexing (uses PostgreSQL indexing instead)
- **Redis** - Lock, concurrency, and persistence
- **External Storage** - S3, Azure Blob (uses database storage)
- **Other Event Queues** - SQS, AMQP, NATS (Kafka only)
- **Other Databases** - Cassandra, MySQL, SQLite (PostgreSQL only)

## Image Size Comparison

| Image Type | Size | Components |
|------------|------|------------|
| **Full** | ~1.2GB | All components + UI + Nginx |
| **Slim** | ~900MB-1GB | All components, no UI |
| **Minimal** | ~550-900MB | Essential components only (PostgreSQL + Kafka) |

**Note**: Spring Boot fat JARs bundle all dependencies (~345MB for minimal), which is the main contributor to image size. The actual runtime image consists of:
- Alpine Linux base: ~8MB
- OpenJDK 17 JRE: ~203MB
- Spring Boot JAR (with all deps): ~345MB
- Config files: ~1MB
- **Total compressed**: ~557MB (~899MB virtual size reported by Docker)

## Architecture

```
┌─────────────────────────────────────────┐
│     Conductor Server (Minimal)          │
│                                          │
│  ┌────────────┐      ┌──────────────┐  │
│  │  REST API  │      │ gRPC Server  │  │
│  └────────────┘      └──────────────┘  │
│           │                 │            │
│  ┌────────────────────────────────────┐ │
│  │      Workflow Engine (Core)        │ │
│  └────────────────────────────────────┘ │
│           │                 │            │
│  ┌────────────────┐  ┌─────────────┐   │
│  │  System Tasks  │  │   Metrics   │   │
│  └────────────────┘  └─────────────┘   │
└─────────────────────────────────────────┘
         │                        │
         ▼                        ▼
┌──────────────────┐    ┌──────────────────┐
│   PostgreSQL     │    │      Kafka       │
│                  │    │                  │
│ - Persistence    │    │ - Event Queues   │
│ - Indexing       │    │ - Event Driven   │
│ - Locking        │    │   Workflows      │
│ - Queues         │    │                  │
└──────────────────┘    └──────────────────┘
```

## Quick Start

### 1. Build the Image

```bash
cd /path/to/conductor
docker build -f docker/server/Dockerfile_Minimal -t conductor:minimal .
```

### 2. Run with Docker Compose

```bash
cd docker
docker-compose -f docker-compose-minimal.yaml up
```

This will start:
- PostgreSQL on port 5432
- Kafka on port 9092
- Conductor Server on port 8080 (REST) and 8090 (gRPC)

### 3. Verify the Setup

```bash
# Check health
curl http://localhost:8080/health

# Check API documentation
curl http://localhost:8080/api-docs

# Prometheus metrics
curl http://localhost:8080/actuator/prometheus
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CONFIG_PROP` | `config-minimal.properties` | Configuration file to use |
| `CONDUCTOR_CONFIG_FILE` | `/app/config/config-minimal.properties` | Full path to config |

### Key Configuration Properties

Located in [config-minimal.properties](config/config-minimal.properties):

```properties
# Database
conductor.db.type=postgres
spring.datasource.url=jdbc:postgresql://postgresdb:5432/postgres

# Distributed Locking (PostgreSQL-based)
conductor.workflow-execution-lock.type=postgres

# Indexing (PostgreSQL-based, no Elasticsearch)
conductor.indexing.enabled=true
conductor.indexing.type=postgres
conductor.elasticsearch.version=0

# Kafka Event Queue
conductor.event-queues.kafka.enabled=true
conductor.default-event-queue.type=kafka
conductor.event-queues.kafka.bootstrapServers=kafka:9092

# Metrics
conductor.metrics-prometheus.enabled=true
```

## Distributed Locking

This minimal setup uses **PostgreSQL-based distributed locking**, which means:
- ✅ Fully supports multi-instance deployments
- ✅ No Redis dependency
- ✅ Uses PostgreSQL's `INSERT ... ON CONFLICT` for atomic lock operations
- ✅ Configurable lease time and retry intervals

Configuration:
```properties
conductor.workflow-execution-lock.type=postgres
conductor.app.workflowExecutionLockEnabled=true
conductor.app.lockTimeToTry=500
```

## External Dependencies

### Required
- **PostgreSQL 12+** - Main database
- **Kafka 2.8+** - Event queue system

### Optional
- None (this is a self-contained minimal setup)

## Use Cases

### ✅ Good For:
- Production deployments with PostgreSQL + Kafka
- Cloud-native environments (Kubernetes, ECS, etc.)
- Microservices architectures
- Event-driven workflows
- Multi-instance distributed deployments
- Resource-constrained environments

### ❌ Not Good For:
- Deployments requiring workflow history search (no Elasticsearch)
- Use cases needing external payload storage (S3, Azure)
- Environments requiring Redis-based features
- Use cases needing the web UI (use full image instead)

## Scaling

### Horizontal Scaling
This minimal image supports horizontal scaling:

```yaml
# docker-compose.yaml
services:
  conductor-server:
    # ... other config
    deploy:
      replicas: 3  # Run 3 instances
```

PostgreSQL distributed locking ensures workflow execution safety across instances.

### Vertical Scaling
Adjust JVM memory:

```yaml
services:
  conductor-server:
    environment:
      - JAVA_OPTS=-Xms512m -Xmx2g
```

## Monitoring

### Prometheus Metrics
Exposed at `http://localhost:8080/actuator/prometheus`

Key metrics:
- `conductor_workflow_execution_total` - Total workflows executed
- `conductor_task_execution_duration_seconds` - Task execution time
- `conductor_queue_size` - Task queue depth

### Health Check
Endpoint: `http://localhost:8080/health`

Checks:
- Database connectivity
- Kafka connectivity
- Application status

## API Access

### REST API
- **Base URL**: `http://localhost:8080/api`
- **Swagger UI**: `http://localhost:8080/swagger-ui.html`
- **OpenAPI Spec**: `http://localhost:8080/api-docs`

### gRPC API
- **Port**: 8090
- **Protocol**: gRPC
- **Proto files**: Available in `grpc` module

## Troubleshooting

### Issue: Database connection fails
**Solution**: Ensure PostgreSQL is healthy
```bash
docker exec conductor-postgres pg_isready -U conductor
```

### Issue: Kafka connection fails
**Solution**: Check Kafka broker status
```bash
docker exec conductor-kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### Issue: Out of memory
**Solution**: Increase JVM heap size
```yaml
environment:
  - JAVA_OPTS=-Xms1g -Xmx4g
```

### Issue: Slow workflow execution
**Solution**: Tune thread pool
```properties
conductor.app.systemTaskWorkerThreadCount=50
conductor.app.systemTaskMaxPollCount=50
```

## Building from Source

### Build server-minimal module only
```bash
# Add server-minimal to settings.gradle (done automatically in Dockerfile)
./gradlew :conductor-server-minimal:build -x test
```

### Check dependencies
```bash
./gradlew :conductor-server-minimal:dependencies --configuration runtimeClasspath
```

## Customization

### Add/Remove Dependencies

Edit [server-minimal/build.gradle](../../server-minimal/build.gradle):

```gradle
dependencies {
    // Add new module
    implementation project(':conductor-new-module')

    // Remove module (comment out)
    // implementation project(':conductor-kafka-event-queue')
}
```

### Change Configuration

Create custom config file in `docker/server/config/`:
```bash
cp config-minimal.properties config-custom.properties
# Edit config-custom.properties
```

Update docker-compose:
```yaml
environment:
  - CONFIG_PROP=config-custom.properties
```

## Production Recommendations

### Security
1. **Use secrets management** for database passwords
2. **Enable SSL** for PostgreSQL and Kafka
3. **Configure authentication** for REST/gRPC APIs
4. **Use network policies** to restrict access

### Performance
1. **Tune PostgreSQL** connection pool:
   ```properties
   spring.datasource.hikari.maximum-pool-size=20
   spring.datasource.hikari.minimum-idle=5
   ```

2. **Optimize Kafka** consumer settings:
   ```properties
   conductor.event-queues.kafka.poll-time-duration=200ms
   ```

3. **Configure thread pools**:
   ```properties
   conductor.app.systemTaskWorkerThreadCount=50
   ```

### High Availability
1. **Run multiple Conductor instances** (3+ recommended)
2. **Use PostgreSQL replication** (primary + replicas)
3. **Use Kafka cluster** (3+ brokers)
4. **Configure health checks** and auto-restart
5. **Monitor metrics** with Prometheus + Grafana

## Migration from Full Image

If migrating from the full Conductor image:

### 1. Export Workflows
```bash
# From full setup
curl http://localhost:8080/api/metadata/workflow > workflows.json
```

### 2. Import to Minimal
```bash
# To minimal setup
curl -X POST http://localhost:8080/api/metadata/workflow \
  -H "Content-Type: application/json" \
  -d @workflows.json
```

### 3. Known Limitations
- **No workflow history search** - Elasticsearch not available
- **No external storage** - Large payloads stored in database
- **No UI** - API-only access

## Files Reference

| File | Purpose |
|------|---------|
| [Dockerfile_Minimal](Dockerfile_Minimal) | Docker build file |
| [config-minimal.properties](config/config-minimal.properties) | Runtime configuration |
| [docker-compose-minimal.yaml](../docker-compose-minimal.yaml) | Docker Compose setup |
| [server-minimal/build.gradle](../../server-minimal/build.gradle) | Module dependencies |

## Image Size Optimization

If the ~900MB image size is a concern, here are options to reduce it:

### Option 1: Use jlink for Custom JRE (Saves ~100-150MB)
Create a minimal JRE with only required modules:
```dockerfile
# In builder stage
RUN jlink \
    --add-modules java.base,java.logging,java.sql,java.naming,java.desktop,java.management,java.security.jgss,java.instrument \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /javaruntime

# In runtime stage - copy custom JRE instead of installing openjdk17-jre
COPY --from=builder /javaruntime $JAVA_HOME
```
**Size reduction**: ~100-150MB (JRE: 203MB → 50-100MB)

### Option 2: Multi-stage Build with Layer Extraction (Saves ~50-100MB)
Extract Spring Boot layers for better caching:
```gradle
// In build.gradle
bootJar {
    layered {
        enabled = true
    }
}
```
```dockerfile
# Extract layers in runtime stage
RUN java -Djarmode=layertools -jar conductor-server.jar extract
# Copy layers separately for better caching
```
**Size reduction**: ~50-100MB through compression and caching

### Option 3: Exclude Unused Metrics Registries (Saves ~20-30MB)
Remove unused metrics implementations from build.gradle:
```gradle
dependencies {
    implementation project(':conductor-metrics')

    // Exclude unused metrics registries
    configurations.all {
        exclude group: 'io.micrometer', module: 'micrometer-registry-atlas'
        exclude group: 'io.micrometer', module: 'micrometer-registry-datadog'
        exclude group: 'io.micrometer', module: 'micrometer-registry-elastic'
        // Keep only prometheus
    }
}
```
**Size reduction**: ~20-30MB

### Option 4: Use Distroless Base Image (Saves ~150MB+)
Replace Alpine + OpenJDK with Google's distroless:
```dockerfile
FROM gcr.io/distroless/java17-debian11:nonroot
```
**Size reduction**: ~150MB+ (smaller base, no package manager overhead)
**Trade-off**: No shell access for debugging

### Option 5: Remove Unnecessary Dependencies
Audit and remove unused dependencies:
```bash
./gradlew :conductor-server-minimal:dependencies --configuration runtimeClasspath | grep -v "(*)"
```
Look for large libraries that aren't actually used.

### Option 6: Switch to Quarkus or Micronaut (Saves ~200-600MB)

**Why Spring Boot is Large:**
- Runtime reflection and classpath scanning
- Large framework overhead (~80-100MB)
- Fat JAR includes all dependencies unoptimized
- No ahead-of-time (AOT) compilation
- Full JDK features required

**Quarkus Benefits:**
- **AOT compilation** - Pre-compiles at build time
- **Native image** support (GraalVM)
- Optimized for containers
- Tree-shaking removes unused code
- **Significantly smaller memory footprint**

**Micronaut Benefits:**
- **Compile-time dependency injection** (no reflection)
- Smaller framework footprint (~30-40MB)
- Fast startup time
- Better suited for microservices

**Size Comparison:**

| Framework | JAR Size | Image Size | Native Image | Startup Time | Memory |
|-----------|----------|------------|--------------|--------------|--------|
| **Spring Boot** (current) | 345MB | 900MB | N/A | ~5-8s | 512MB+ |
| **Micronaut** | 80-120MB | 300-400MB | 50-80MB | ~1-2s | 256MB |
| **Quarkus JVM** | 60-100MB | 250-350MB | - | ~1-2s | 256MB |
| **Quarkus Native** | N/A | **80-150MB** | 30-50MB | ~0.05s | 128MB |

**Quarkus Native Image Example:**
```dockerfile
# Quarkus native build
FROM quay.io/quarkus/ubi-quarkus-native-image:22.3-java17 AS builder
WORKDIR /app
COPY . .
RUN ./gradlew build -Dquarkus.package.type=native

# Minimal runtime
FROM quay.io/quarkus/quarkus-micro-image:2.0
COPY --from=builder /app/build/*-runner /application
EXPOSE 8080
CMD ["./application"]
```
**Final size: 80-150MB** (10x smaller!)

**Why Not Use Quarkus/Micronaut Now?**

The current Conductor codebase is tightly coupled to Spring Boot:
- Heavy use of Spring annotations (`@Autowired`, `@Service`, `@Configuration`)
- Spring Data repositories
- Spring MVC controllers
- Spring Boot actuator
- Spring's dependency injection throughout

**Effort Required to Migrate:**
- **High effort**: 2-4 weeks of development
- Rewrite all Spring annotations → Quarkus/Micronaut equivalents
- Replace Spring Data with Panache (Quarkus) or Micronaut Data
- Rewrite REST controllers (different annotations)
- Replace Spring Boot actuator with framework-specific health checks
- Extensive integration testing

**Would It Be Worth It?**

| Benefit | Spring Boot | Quarkus Native | Savings |
|---------|-------------|----------------|---------|
| Image size | 900MB | 80-150MB | **85-90%** |
| Memory usage | 512MB+ | 128MB | **75%** |
| Startup time | 5-8s | 0.05s | **99%** |
| Cold start | Slow | Instant | **95%+** |
| Monthly cloud cost (example) | $100 | $20-30 | **70-80%** |

**When to Consider Migration:**
- ✅ Running 10+ instances in production (significant cost savings)
- ✅ Serverless/FaaS deployments (cold start critical)
- ✅ Kubernetes with frequent scaling (fast startup matters)
- ✅ Resource-constrained environments
- ❌ Single instance / development only (not worth effort)
- ❌ Limited development resources

**Recommendation:**
- **For production at scale**: Migrating to Quarkus native would provide **70-85% cost savings**
- **For development/small deployments**: Current Spring Boot is perfectly fine
- **Best approach**: Create a parallel `server-quarkus` module, maintain both versions

**Estimated Image Size with Quarkus Native:**
```
Native executable:     30-50MB
UBI Micro base:        30-40MB
Config files:           1MB
──────────────────────────
Total:                 60-90MB (90% reduction!)
```

### Recommended Approach
**Best balance of size vs maintainability:**
1. Use jlink for custom JRE (-150MB)
2. Enable Spring Boot layers (-50MB)
3. Exclude unused metrics registries (-20MB)

**Expected final size**: ~580MB (35% reduction from 900MB)

For the absolute smallest image:
- Use distroless base + jlink + layer optimization: **~350-400MB**
- Trade-off: No debugging tools, harder to troubleshoot

## Support

- **Documentation**: https://conductor.netflix.com/
- **GitHub Issues**: https://github.com/conductor-oss/conductor/issues
- **Community**: https://conductor.netflix.com/community/

## License

Apache License 2.0 - See LICENSE file
