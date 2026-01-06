# Scaling Considerations

<cite>
**Referenced Files in This Document**   
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py)
- [utils.py](file://mlflow/store/db/utils.py)
- [environment_variables.py](file://mlflow/environment_variables.py)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py)
- [db.py](file://mlflow/db.py)
- [backend-store.mdx](file://docs/docs/self-hosting/architecture/backend-store.mdx)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Database Backend Selection](#database-backend-selection)
3. [Connection Pooling Configuration](#connection-pooling-configuration)
4. [Horizontal Scaling Strategies](#horizontal-scaling-strategies)
5. [Tracking Server Architecture](#tracking-server-architecture)
6. [REST API Handlers and Database Store](#rest-api-handlers-and-database-store)
7. [SQLAlchemy Store Implementation](#sqlalchemy-store-implementation)
8. [Relationships Between Components](#relationships-between-components)
9. [Common Scaling Issues](#common-scaling-issues)
10. [Monitoring and Capacity Planning](#monitoring-and-capacity-planning)
11. [Disaster Recovery](#disaster-recovery)
12. [Conclusion](#conclusion)

## Introduction
Scaling MLflow to support large teams and high-volume workloads requires careful consideration of database backend selection, connection pooling, and horizontal scaling strategies. This document provides comprehensive guidance on implementing scalable MLflow deployments, focusing on the architecture of the tracking server and its components that impact scalability. The content is designed to be accessible to beginners while providing sufficient technical depth for experienced developers managing production deployments.

## Database Backend Selection
MLflow supports multiple database backends through SQLAlchemy, enabling flexibility in choosing the appropriate database system for different deployment scenarios. The choice of database backend significantly impacts scalability, performance, and reliability.

For production deployments with large teams and high-volume workloads, relational databases such as PostgreSQL, MySQL, and Microsoft SQL Server are recommended over the default SQLite backend. These databases provide better concurrency handling, connection pooling, and transaction isolation capabilities essential for scaling.

The database backend is configured through the tracking URI when starting the MLflow server. For example:
- PostgreSQL: `postgresql://user:password@host:port/database`
- MySQL: `mysql+pymysql://user:password@host:port/database`
- Microsoft SQL Server: `mssql+pymssql://user:password@host:port/database`

SQLite is suitable for development and small-scale deployments but has limitations in concurrent write operations and connection handling that make it unsuitable for production environments with multiple users and high write volumes.

**Section sources**
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1-L50)
- [utils.py](file://mlflow/store/db/utils.py#L1-L50)

## Connection Pooling Configuration
Connection pooling is a critical aspect of scaling MLflow deployments, as it manages database connections efficiently and reduces the overhead of establishing new connections for each request.

MLflow provides several environment variables to configure connection pooling options for the SQLAlchemy store:

```mermaid
erDiagram
MLFLOW_SQLALCHEMYSTORE_POOL_SIZE ||--o{ ConnectionPool : "sets"
MLFLOW_SQLALCHEMYSTORE_MAX_OVERFLOW ||--o{ ConnectionPool : "sets"
MLFLOW_SQLALCHEMYSTORE_POOL_RECYCLE ||--o{ ConnectionPool : "sets"
MLFLOW_SQLALCHEMYSTORE_POOLCLASS ||--o{ ConnectionPool : "sets"
ConnectionPool ||--o{ Database : "connects to"
```

**Diagram sources**
- [utils.py](file://mlflow/store/db/utils.py#L225-L267)
- [environment_variables.py](file://mlflow/environment_variables.py#L225-L267)

The key connection pooling parameters include:
- `MLFLOW_SQLALCHEMYSTORE_POOL_SIZE`: Sets the minimum number of connections to maintain in the pool
- `MLFLOW_SQLALCHEMYSTORE_MAX_OVERFLOW`: Specifies the maximum number of connections that can be created beyond the pool size
- `MLFLOW_SQLALCHEMYSTORE_POOL_RECYCLE`: Determines how frequently connections are recycled to prevent staleness
- `MLFLOW_SQLALCHEMYSTORE_POOLCLASS`: Allows selection of different pool implementations (QueuePool, NullPool, etc.)

Proper configuration of these parameters is essential for handling concurrent requests efficiently. For high-traffic deployments, increasing the pool size and overflow values can help accommodate more simultaneous connections, while setting an appropriate recycle time prevents connection timeouts in long-running deployments.

**Section sources**
- [utils.py](file://mlflow/store/db/utils.py#L332-L369)
- [environment_variables.py](file://mlflow/environment_variables.py#L225-L267)

## Horizontal Scaling Strategies
Horizontal scaling of MLflow involves deploying multiple tracking server instances behind a load balancer to distribute incoming requests and improve availability.

The MLflow tracking server is stateless, making it well-suited for horizontal scaling. Each server instance connects to the same backend database and artifact store, allowing seamless distribution of requests across multiple instances.

Key considerations for horizontal scaling include:
- Using a shared database backend that can handle concurrent connections from multiple server instances
- Configuring a shared artifact store accessible to all server instances
- Implementing a load balancer with appropriate health checks and session persistence
- Ensuring consistent configuration across all server instances

For cloud deployments, MLflow can be deployed using container orchestration platforms like Kubernetes, which provide built-in load balancing, auto-scaling, and service discovery capabilities.

```mermaid
graph TB
subgraph "Load Balancer"
LB[Load Balancer]
end
subgraph "MLflow Servers"
S1[MLflow Server 1]
S2[MLflow Server 2]
S3[MLflow Server 3]
end
subgraph "Shared Storage"
DB[(Database)]
AS[(Artifact Store)]
end
LB --> S1
LB --> S2
LB --> S3
S1 --> DB
S1 --> AS
S2 --> DB
S2 --> AS
S3 --> DB
S3 --> AS
```

**Diagram sources**
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1-L50)

**Section sources**
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1-L50)

## Tracking Server Architecture
The MLflow tracking server architecture consists of several components that work together to handle requests, store metadata, and serve artifacts.

The server is implemented as a FastAPI application that wraps the existing Flask application using WSGIMiddleware, maintaining API compatibility while enabling future migration to native FastAPI endpoints.

```mermaid
graph TD
Client[Client Application] --> LB[Load Balancer]
LB --> FastAPI[FastAPI Application]
FastAPI --> Flask[Flask Application]
FastAPI --> OTEL[OpenTelemetry API]
FastAPI --> JobAPI[Job API]
FastAPI --> GatewayAPI[Gateway API]
Flask --> SQLAlchemyStore[SQLAlchemy Store]
SQLAlchemyStore --> Database[(Database)]
Flask --> ArtifactStore[Artifact Store]
ArtifactStore --> Storage[(Storage Backend)]
```

**Diagram sources**
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1-L50)

The architecture includes:
- FastAPI application: Provides the main entry point and routes requests to appropriate handlers
- WSGIMiddleware: Wraps the Flask application to maintain API compatibility
- REST API handlers: Process incoming requests and interact with the backend store
- SQLAlchemy store: Manages database operations and transactions
- Artifact store: Handles storage and retrieval of large artifacts

This layered architecture allows for independent scaling of different components and provides flexibility in deployment configurations.

**Section sources**
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1-L50)

## REST API Handlers and Database Store
The REST API handlers in MLflow are responsible for processing incoming requests and interacting with the database store to perform CRUD operations on experiments, runs, metrics, and other entities.

The API handlers use the SQLAlchemy store to manage database operations, ensuring proper transaction handling and error recovery. Each handler method typically follows a pattern of:
1. Validating input parameters
2. Creating a database session
3. Performing the requested operation within a transaction
4. Committing the transaction or rolling back on error
5. Returning the result or appropriate error response

The managed session maker pattern is used to ensure that database sessions are properly managed and closed, even in the presence of exceptions. This pattern wraps session creation in a context manager that automatically handles commit, rollback, and cleanup operations.

```mermaid
sequenceDiagram
participant Client as "Client App"
participant Handler as "REST API Handler"
participant Store as "SQLAlchemy Store"
participant Session as "Database Session"
participant DB as "Database"
Client->>Handler : API Request
Handler->>Store : Operation Request
Store->>Session : Create Session
Session->>DB : Execute Query
DB-->>Session : Query Result
Session->>Store : Return Result
Store->>Handler : Operation Result
Handler->>Client : API Response
alt Error Occurs
Session->>Session : Rollback Transaction
Store->>Handler : Error Response
Handler->>Client : Error Response
end
```

**Diagram sources**
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L150-L205)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)

**Section sources**
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L150-L205)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)

## SQLAlchemy Store Implementation
The SQLAlchemy store is the core component responsible for managing database operations in MLflow. It provides an abstraction layer over the underlying database, handling connection management, transaction handling, and query execution.

The store implementation includes several key features that impact scalability:

### Connection Management
The store uses SQLAlchemy's connection pooling to manage database connections efficiently. Connection parameters can be configured through environment variables, allowing fine-tuning for different deployment scenarios.

### Transaction Handling
The store implements a managed session pattern that ensures proper transaction handling. Each operation is wrapped in a transaction that is automatically committed on success or rolled back on error.

### Query Optimization
The store includes optimizations for handling large datasets and high-volume workloads. For example, metric logging operations batch metric keys into groups of 500 to avoid binding too many parameters to SQL queries, which can cause performance issues or limit exceeded errors on certain database platforms.

```python
# Example from codebase: batching metric keys
metric_key_batches = [metric_keys[i : i + 500] for i in range(0, len(metric_keys), 500)]
```

### Error Handling and Retry Logic
The store includes retry logic for handling transient database errors. The `create_sqlalchemy_engine_with_retry` function implements exponential backoff to handle temporary connection issues.

```mermaid
flowchart TD
Start([Create Engine]) --> CheckConnection["Check Connection"]
CheckConnection --> ConnectionSuccess{"Connection Successful?"}
ConnectionSuccess --> |Yes| ReturnEngine["Return Engine"]
ConnectionSuccess --> |No| CheckAttempts{"Max Attempts Reached?"}
CheckAttempts --> |No| Wait["Wait with Exponential Backoff"]
Wait --> CheckConnection
CheckAttempts --> |Yes| RaiseError["Raise Exception"]
ReturnEngine --> End([Engine Created])
RaiseError --> End
```

**Diagram sources**
- [utils.py](file://mlflow/store/db/utils.py#L309-L329)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1029-L1051)

**Section sources**
- [utils.py](file://mlflow/store/db/utils.py#L309-L329)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1029-L1051)

## Relationships Between Components
The relationships between the tracking server, backend databases, and artifact stores in distributed environments are critical for understanding MLflow's scalability characteristics.

```mermaid
graph TD
subgraph "Client Applications"
C1[Client 1]
C2[Client 2]
C3[Client 3]
end
subgraph "MLflow Servers"
S1[Server 1]
S2[Server 2]
S3[Server 3]
end
subgraph "Storage Layer"
DB[(Database)]
AS[(Artifact Store)]
end
C1 --> S1
C2 --> S2
C3 --> S3
S1 --> DB
S1 --> AS
S2 --> DB
S2 --> AS
S3 --> DB
S3 --> AS
```

**Diagram sources**
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1-L50)

In distributed environments:
- Multiple client applications connect to one or more MLflow servers
- Each server instance connects to the same shared database and artifact store
- The database stores metadata about experiments, runs, metrics, and parameters
- The artifact store stores large files such as models, images, and datasets

This architecture allows for horizontal scaling of both client and server components while maintaining data consistency through the shared storage layer.

**Section sources**
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L1-L50)

## Common Scaling Issues
Several common issues can arise when scaling MLflow deployments, particularly related to database performance and concurrency.

### Database Lock Contention
Database lock contention occurs when multiple transactions attempt to modify the same rows simultaneously. MLflow addresses this through:
- Using write locks on rows during metric updates
- Ordering operations consistently to reduce deadlock likelihood
- Batching operations to minimize transaction duration

### Slow Query Performance
Slow query performance can result from:
- Large result sets without proper pagination
- Missing indexes on frequently queried columns
- Complex joins across multiple tables

MLflow implements pagination for large result sets and uses appropriate indexing strategies to optimize query performance.

### Migration Challenges
Upgrading large MLflow instances can be challenging due to:
- Schema migration time for large databases
- Downtime requirements during migration
- Compatibility issues between versions

The `mlflow db upgrade` command provides a mechanism for upgrading the database schema, but administrators should plan for potential downtime and test migrations on a copy of the production database first.

**Section sources**
- [db.py](file://mlflow/db.py#L1-L28)
- [utils.py](file://mlflow/store/db/utils.py#L262-L289)

## Monitoring and Capacity Planning
Effective monitoring and capacity planning are essential for maintaining scalable MLflow deployments.

### Monitoring System Health
Key metrics to monitor include:
- Database connection pool usage
- Query response times
- Server CPU and memory utilization
- Artifact storage usage
- API request rates and error rates

MLflow provides built-in support for exporting metrics to monitoring systems, and administrators should implement alerting for critical thresholds.

### Capacity Planning
Capacity planning should consider:
- Expected growth in the number of experiments and runs
- Storage requirements for artifacts
- Network bandwidth for artifact transfers
- Database size and performance requirements

Regular capacity assessments help ensure that the infrastructure can handle current and future workloads without performance degradation.

**Section sources**
- [environment_variables.py](file://mlflow/environment_variables.py#L505-L523)
- [system_metrics](file://mlflow/system_metrics)

## Disaster Recovery
Disaster recovery planning is critical for production MLflow deployments to ensure data availability and minimize downtime.

### Backup Strategies
Regular backups of both the database and artifact store are essential. Backup frequency should be determined by the rate of data change and recovery point objectives.

### High Availability
Implementing high availability configurations, such as database replication and multiple server instances, helps ensure service continuity during hardware failures or maintenance.

### Recovery Procedures
Documented recovery procedures should include:
- Steps for restoring from backups
- Verification of data integrity after recovery
- Communication protocols for stakeholders

Regular testing of recovery procedures ensures they work as expected when needed.

**Section sources**
- [db.py](file://mlflow/db.py#L1-L28)
- [utils.py](file://mlflow/store/db/utils.py#L262-L289)

## Conclusion
Scaling MLflow to support large teams and high-volume workloads requires careful consideration of database backend selection, connection pooling, and horizontal scaling strategies. By understanding the architecture of the tracking server and its components, administrators can implement effective scaling solutions that meet their organization's needs. Proper monitoring, capacity planning, and disaster recovery planning are essential for maintaining reliable and performant MLflow deployments in production environments.