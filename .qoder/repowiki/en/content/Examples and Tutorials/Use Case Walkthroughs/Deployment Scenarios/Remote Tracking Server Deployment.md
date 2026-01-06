# Remote Tracking Server Deployment

<cite>
**Referenced Files in This Document**   
- [server.py](file://dev/server.py)
- [handlers.py](file://mlflow/server/handlers.py)
- [__init__.py](file://mlflow/server/__init__.py)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py)
- [security.py](file://mlflow/server/security.py)
- [auth/__init__.py](file://mlflow/server/auth/__init__.py)
- [constants.py](file://mlflow/server/constants.py)
- [db.py](file://mlflow/db.py)
- [utils.py](file://mlflow/store/db/utils.py)
- [file_store.py](file://mlflow/store/tracking/file_store.py)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py)
- [artifact_repository_registry.py](file://mlflow/store/artifact/artifact_repository_registry.py)
- [environment_variables.py](file://mlflow/environment_variables.py)
- [rest_utils.py](file://mlflow/utils/rest_utils.py)
- [README.md](file://docs/docs/self-hosting/architecture/backend-store.mdx)
- [tracking-server.mdx](file://docs/docs/self-hosting/architecture/tracking-server.mdx)
- [custom.md](file://docs/docs/self-hosting/security/custom.md)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Conclusion](#conclusion)

## Introduction
This document provides comprehensive guidance on deploying MLflow with remote tracking servers. It covers the implementation details of setting up and configuring MLflow tracking servers for distributed environments, including backend stores, artifact repositories, authentication mechanisms, and high availability configurations. The document explains how to secure tracking servers with HTTPS, implement user authentication and authorization, and configure enterprise-grade deployments with proper monitoring and disaster recovery procedures.

## Project Structure
The MLflow repository contains a well-organized structure for the tracking server components, with clear separation between server logic, storage backends, and security mechanisms. The core server components are located in the `mlflow/server/` directory, while storage implementations are organized in `mlflow/store/` with subdirectories for tracking and artifact storage. Configuration and environment variables are managed through dedicated modules, and authentication mechanisms are implemented as pluggable components.

```mermaid
graph TD
subgraph "Server Core"
A[server/__init__.py]
B[server/handlers.py]
C[server/fastapi_app.py]
end
subgraph "Storage Components"
D[store/tracking/]
E[store/artifact/]
F[store/db/]
end
subgraph "Security & Authentication"
G[server/auth/]
H[server/security.py]
end
subgraph "Configuration"
I[environment_variables.py]
J[server/constants.py]
end
A --> B
A --> C
G --> A
H --> A
D --> A
E --> A
F --> D
I --> A
J --> A
```

**Diagram sources**
- [__init__.py](file://mlflow/server/__init__.py)
- [handlers.py](file://mlflow/server/handlers.py)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py)
- [auth/__init__.py](file://mlflow/server/auth/__init__.py)
- [security.py](file://mlflow/server/security.py)
- [file_store.py](file://mlflow/store/tracking/file_store.py)
- [artifact_repository_registry.py](file://mlflow/store/artifact/artifact_repository_registry.py)
- [db.py](file://mlflow/db.py)
- [constants.py](file://mlflow/server/constants.py)
- [environment_variables.py](file://mlflow/environment_variables.py)

**Section sources**
- [__init__.py](file://mlflow/server/__init__.py)
- [handlers.py](file://mlflow/server/handlers.py)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py)

## Core Components
The MLflow tracking server architecture consists of several core components that work together to provide a robust model tracking solution. The server implementation supports both Flask and FastAPI frameworks, with the FastAPI application wrapping the Flask app for backward compatibility. The handlers module contains all endpoint implementations, while the security module provides middleware for host validation, CORS protection, and security headers. Authentication is implemented as a pluggable system with support for basic authentication and custom authorization functions.

The backend store configuration is managed through environment variables and command-line options, allowing flexible deployment configurations. The server supports multiple database backends through SQLAlchemy and various artifact repository types including cloud storage providers. The architecture is designed to be extensible, with plugin points for custom authentication, authorization, and storage implementations.

**Section sources**
- [__init__.py](file://mlflow/server/__init__.py#L1-L473)
- [handlers.py](file://mlflow/server/handlers.py#L1-L5111)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L1-L63)
- [security.py](file://mlflow/server/security.py#L1-L116)

## Architecture Overview
The MLflow tracking server follows a modular architecture with clear separation of concerns between the web server layer, business logic, and data storage components. The server can be deployed with various WSGI servers including uvicorn (default), gunicorn, and waitress, with configuration options for worker processes and server-specific parameters.

```mermaid
graph TD
Client[MLflow Client] --> |HTTP/HTTPS| LoadBalancer[Load Balancer]
LoadBalancer --> Server1[MLflow Server Instance 1]
LoadBalancer --> Server2[MLflow Server Instance 2]
LoadBalancer --> ServerN[MLflow Server Instance N]
subgraph "Server Components"
Server1 --> Flask[Flask Application]
Server2 --> Flask
ServerN --> Flask
Flask --> FastAPI[FastAPI Wrapper]
FastAPI --> Handlers[Endpoint Handlers]
Handlers --> BackendStore[Backend Store]
Handlers --> ArtifactRepo[Artifact Repository]
Handlers --> Auth[Authentication]
end
BackendStore --> Database[(SQLAlchemy Database)]
ArtifactRepo --> S3[(S3 Bucket)]
ArtifactRepo --> GCS[(Google Cloud Storage)]
ArtifactRepo --> Azure[(Azure Blob Storage)]
Auth --> UserDB[(User Database)]
style Server1 fill:#f9f,stroke:#333
style Server2 fill:#f9f,stroke:#333
style ServerN fill:#f9f,stroke:#333
```

**Diagram sources**
- [__init__.py](file://mlflow/server/__init__.py#L309-L473)
- [handlers.py](file://mlflow/server/handlers.py#L531-L572)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py#L21-L62)
- [security.py](file://mlflow/server/security.py#L38-L116)

## Detailed Component Analysis

### Backend Store Configuration
The backend store in MLflow stores metadata for runs, models, traces, and experiments including run IDs, model IDs, trace IDs, tags, timestamps, parameters, and metrics. MLflow supports two primary types of backend stores:

1. **Relational Database (Default)**: MLflow supports various databases through SQLAlchemy, including SQLite (default), PostgreSQL, MySQL, and Microsoft SQL Server. This option provides better performance through indexing and is easier to scale to larger volumes of data.

2. **Local File System (Legacy)**: The file-based backend stores metadata in local files in the `./mlruns` directory. This was the default backend in earlier versions but is now in maintenance mode and not recommended for new deployments.

The backend store URI is configured using the `--backend-store-uri` command-line option when starting the server or via the `MLFLOW_TRACKING_URI` environment variable. For database backends, the URI follows the SQLAlchemy format (e.g., `postgresql://user:password@host:port/database`).

```mermaid
classDiagram
class BackendStore {
+initialize_backend_stores(backend_store_uri, registry_store_uri, default_artifact_root)
+_get_tracking_store(backend_store_uri, default_artifact_root)
+_get_model_registry_store(registry_store_uri)
+_get_artifact_repo(artifact_uri)
}
class SQLAlchemyStore {
+get_run(run_id)
+create_experiment(name)
+log_metric(run_id, metric)
+log_param(run_id, param)
+set_tag(run_id, tag)
}
class FileStore {
+get_run(run_id)
+create_experiment(name)
+log_metric(run_id, metric)
+log_param(run_id, param)
+set_tag(run_id, tag)
}
BackendStore <|-- SQLAlchemyStore
BackendStore <|-- FileStore
SQLAlchemyStore --> "SQLAlchemy" Database
FileStore --> "Local File System" FileSystem
```

**Diagram sources**
- [handlers.py](file://mlflow/server/handlers.py#L531-L572)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py#L757-L781)
- [file_store.py](file://mlflow/store/tracking/file_store.py#L214-L240)
- [README.md](file://docs/docs/self-hosting/architecture/backend-store.mdx)

**Section sources**
- [handlers.py](file://mlflow/server/handlers.py#L531-L572)
- [sqlalchemy_store.py](file://mlflow/store/tracking/sqlalchemy_store.py)
- [file_store.py](file://mlflow/store/tracking/file_store.py)
- [README.md](file://docs/docs/self-hosting/architecture/backend-store.mdx)

### Artifact Repository Configuration
MLflow supports various artifact repositories for storing large model artifacts such as model weight files. The artifact repository is configured using the `--default-artifact-root` command-line option or the `MLFLOW_ARTIFACT_ROOT` environment variable.

The supported artifact repository types include:
- **Amazon S3**: `s3://bucket/path`
- **Google Cloud Storage**: `gs://bucket/path`
- **Azure Blob Storage**: `wasbs://container@account.blob.core.windows.net/path` or `abfss://container@account.blob.core.windows.net/path`
- **Local File System**: `file:/path` or `/absolute/path`
- **FTP Server**: `ftp://user:password@host/path`
- **SFTP Server**: `sftp://user:password@host/path`
- **HDFS**: `hdfs://host:port/path`
- **HTTP Server**: `http://host:port/path` or `https://host:port/path`

The artifact repository system uses a registry pattern to map URI schemes to specific repository implementations. When an artifact operation is requested, MLflow determines the appropriate repository based on the URI scheme and delegates the operation to that repository.

```mermaid
classDiagram
class ArtifactRepository {
+list_artifacts(path)
+download_artifacts(artifact_path, dst_path)
+log_artifact(local_file, artifact_path)
+log_artifacts(local_dir, artifact_path)
+delete_artifacts(artifact_path)
}
class S3ArtifactRepository {
+list_artifacts(path)
+download_artifacts(artifact_path, dst_path)
+log_artifact(local_file, artifact_path)
+log_artifacts(local_dir, artifact_path)
}
class GCSArtifactRepository {
+list_artifacts(path)
+download_artifacts(artifact_path, dst_path)
+log_artifact(local_file, artifact_path)
+log_artifacts(local_dir, artifact_path)
}
class AzureBlobArtifactRepository {
+list_artifacts(path)
+download_artifacts(artifact_path, dst_path)
+log_artifact(local_file, artifact_path)
+log_artifacts(local_dir, artifact_path)
}
class LocalArtifactRepository {
+list_artifacts(path)
+download_artifacts(artifact_path, dst_path)
+log_artifact(local_file, artifact_path)
+log_artifacts(local_dir, artifact_path)
}
ArtifactRepository <|-- S3ArtifactRepository
ArtifactRepository <|-- GCSArtifactRepository
ArtifactRepository <|-- AzureBlobArtifactRepository
ArtifactRepository <|-- LocalArtifactRepository
class ArtifactRepositoryRegistry {
-registry : Map[string, ArtifactRepository]
+register(scheme, repository_class)
+get_repository(artifact_uri)
}
ArtifactRepositoryRegistry --> ArtifactRepository
```

**Diagram sources**
- [artifact_repository_registry.py](file://mlflow/store/artifact/artifact_repository_registry.py#L1-L125)
- [utils.py](file://mlflow/utils/_unity_catalog_utils.py#L234-L268)
- [unity_catalog_oss_models_artifact_repo.py](file://mlflow/store/artifact/unity_catalog_oss_models_artifact_repo.py#L137-L168)

**Section sources**
- [artifact_repository_registry.py](file://mlflow/store/artifact/artifact_repository_registry.py)
- [utils.py](file://mlflow/utils/_unity_catalog_utils.py)
- [unity_catalog_oss_models_artifact_repo.py](file://mlflow/store/artifact/unity_catalog_oss_models_artifact_repo.py)

### Authentication and Authorization
MLflow provides a flexible authentication and authorization system that can be enabled by specifying an app name when starting the server. The basic authentication system is enabled with `--app-name basic-auth` and supports user management, password hashing, and permission-based access control.

The authorization system implements role-based access control with four permission levels:
- **READ**: View experiments, runs, and models
- **UPDATE**: Create and modify runs, log metrics and parameters
- **DELETE**: Delete experiments, runs, and models
- **MANAGE**: Full control including managing permissions

Permissions are assigned at the experiment and registered model levels, with inheritance for child resources. The system supports both declarative configuration via INI files and programmatic configuration through custom authorization functions.

```mermaid
classDiagram
class AuthConfig {
+default_permission : Permission
+authorization_function : str
}
class User {
+username : str
+password_hash : str
+is_admin : bool
+created_time : datetime
}
class Permission {
+READ : Permission
+UPDATE : Permission
+DELETE : Permission
+MANAGE : Permission
+can_read : bool
+can_update : bool
+can_delete : bool
+can_manage : bool
}
class ExperimentPermission {
+experiment_id : str
+username : str
+permission : Permission
}
class RegisteredModelPermission {
+registered_model_name : str
+username : str
+permission : Permission
}
class ScorerPermission {
+experiment_id : str
+scorer_name : str
+username : str
+permission : Permission
}
class SqlAlchemyStore {
+create_user(username, password, is_admin)
+get_user(username)
+create_experiment_permission(experiment_id, username, permission)
+get_experiment_permission(experiment_id, username)
+list_experiment_permissions(username)
+create_registered_model_permission(name, username, permission)
+get_registered_model_permission(name, username)
}
User "1" --> "*" ExperimentPermission
User "1" --> "*" RegisteredModelPermission
User "1" --> "*" ScorerPermission
Permission "1" --> "*" ExperimentPermission
Permission "1" --> "*" RegisteredModelPermission
Permission "1" --> "*" ScorerPermission
SqlAlchemyStore --> User
SqlAlchemyStore --> ExperimentPermission
SqlAlchemyStore --> RegisteredModelPermission
SqlAlchemyStore --> ScorerPermission
```

**Diagram sources**
- [auth/__init__.py](file://mlflow/server/auth/__init__.py#L1-L800)
- [custom.md](file://docs/docs/self-hosting/security/custom.md#L108-L118)

**Section sources**
- [auth/__init__.py](file://mlflow/server/auth/__init__.py)
- [custom.md](file://docs/docs/self-hosting/security/custom.md)

### Server Configuration and Deployment
The MLflow tracking server is configured through a combination of command-line arguments and environment variables. The server can be started with various options to control its behavior, including host, port, number of workers, and server-specific options for gunicorn, waitress, or uvicorn.

Key configuration options include:
- `--backend-store-uri`: URI for the backend store (e.g., `sqlite:///mlflow.db`, `postgresql://...`)
- `--default-artifact-root`: Default root directory for storing run artifacts
- `--host`: Host to bind to (default: 127.0.0.1)
- `--port`: Port to listen on (default: 5000)
- `--workers`: Number of worker processes (default: 4)
- `--app-name`: Name of the authentication app to enable
- `--env-file`: Environment file to load configuration from

The server also supports environment variables for configuration, including `MLFLOW_TRACKING_URI`, `MLFLOW_REGISTRY_URI`, and various server-specific variables prefixed with `_MLFLOW_SERVER_`.

```mermaid
flowchart TD
Start([Start MLflow Server]) --> ParseArgs["Parse Command Line Arguments"]
ParseArgs --> CheckEnv["Check Environment Variables"]
CheckEnv --> ConfigStore["Configure Backend Store"]
ConfigStore --> ConfigArtifacts["Configure Artifact Repository"]
ConfigArtifacts --> SetupSecurity["Setup Security Middleware"]
SetupSecurity --> InitAuth["Initialize Authentication"]
InitAuth --> StartServer["Start Server Process"]
StartServer --> Monitor["Monitor Server Health"]
Monitor --> End([Server Running])
subgraph "Configuration Sources"
CmdLine["Command Line Arguments"]
EnvVars["Environment Variables"]
ConfigFile["Configuration File"]
end
CmdLine --> ParseArgs
EnvVars --> CheckEnv
ConfigFile --> CheckEnv
```

**Diagram sources**
- [__init__.py](file://mlflow/server/__init__.py#L309-L473)
- [constants.py](file://mlflow/server/constants.py#L1-L31)
- [environment_variables.py](file://mlflow/environment_variables.py#L75-L107)
- [server.py](file://dev/server.py)

**Section sources**
- [__init__.py](file://mlflow/server/__init__.py)
- [constants.py](file://mlflow/server/constants.py)
- [environment_variables.py](file://mlflow/environment_variables.py)
- [server.py](file://dev/server.py)

## Dependency Analysis
The MLflow tracking server has a well-defined dependency structure with clear separation between core server functionality, storage backends, and security components. The server core depends on Flask for web handling and FastAPI for modern API support, with WSGI middleware to integrate both frameworks.

The storage layer has dependencies on SQLAlchemy for database operations, various cloud storage SDKs for artifact repositories (boto3 for S3, google-cloud-storage for GCS, azure-storage-blob for Azure), and requests for HTTP-based repositories. The authentication system depends on Flask-WTF for CSRF protection and passlib for password hashing.

```mermaid
graph TD
subgraph "Web Framework"
Flask[Flask]
FastAPI[FastAPI]
WSGI[WSGIMiddleware]
end
subgraph "Storage Dependencies"
SQLAlchemy[SQLAlchemy]
Boto3[boto3]
GoogleCloud[google-cloud-storage]
AzureStorage[azure-storage-blob]
Requests[requests]
HDFS[hdfs]
end
subgraph "Security Dependencies"
FlaskWTF[Flask-WTF]
Passlib[passlib]
FlaskCORS[Flask-CORS]
end
subgraph "Monitoring Dependencies"
Prometheus[prometheus-flask-exporter]
Huey[huey]
end
Flask --> Handlers[handlers.py]
FastAPI --> FastAPIApp[fastapi_app.py]
WSGI --> FastAPIApp
FastAPIApp --> Flask
Handlers --> SQLAlchemy
Handlers --> Boto3
Handlers --> GoogleCloud
Handlers --> AzureStorage
Handlers --> Requests
Handlers --> HDFS
Security[security.py] --> FlaskCORS
Auth[auth/__init__.py] --> FlaskWTF
Auth --> Passlib
PrometheusExporter[prometheus_exporter.py] --> Prometheus
JobRunner[job_api.py] --> Huey
```

**Diagram sources**
- [__init__.py](file://mlflow/server/__init__.py)
- [handlers.py](file://mlflow/server/handlers.py)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py)
- [security.py](file://mlflow/server/security.py)
- [auth/__init__.py](file://mlflow/server/auth/__init__.py)
- [prometheus_exporter.py](file://mlflow/server/prometheus_exporter.py)

**Section sources**
- [__init__.py](file://mlflow/server/__init__.py)
- [handlers.py](file://mlflow/server/handlers.py)
- [fastapi_app.py](file://mlflow/server/fastapi_app.py)
- [security.py](file://mlflow/server/security.py)
- [auth/__init__.py](file://mlflow/server/auth/__init__.py)

## Performance Considerations
When deploying MLflow tracking servers in production environments, several performance considerations should be addressed:

1. **Database Connection Pooling**: Configure appropriate connection pool settings using environment variables:
   - `MLFLOW_SQLALCHEMYSTORE_POOL_SIZE`: Number of connections to maintain in the pool
   - `MLFLOW_SQLALCHEMYSTORE_MAX_OVERFLOW`: Maximum number of connections to create beyond the pool size
   - `MLFLOW_SQLALCHEMYSTORE_POOL_RECYCLE`: Number of seconds after which connections are recycled
   - `MLFLOW_SQLALCHEMYSTORE_POOLCLASS`: Connection pool class to use

2. **Server Timeout Configuration**: Adjust server timeouts to handle long-running artifact uploads:
   - For uvicorn: `--uvicorn-opts "--timeout-keep-alive=120"`
   - For gunicorn: `--gunicorn-opts "--timeout=120"`

3. **High Availability**: Deploy multiple server instances behind a load balancer with a shared database and artifact storage.

4. **Monitoring**: Enable Prometheus metrics export with `--expose-prometheus` to monitor server performance and health.

5. **Artifact Storage Costs**: Consider using tiered storage strategies and lifecycle policies to manage costs, especially with cloud storage providers.

**Section sources**
- [utils.py](file://mlflow/store/db/utils.py#L332-L369)
- [tracking-server.mdx](file://docs/docs/self-hosting/architecture/tracking-server.mdx#L455-L465)
- [prometheus_exporter.py](file://mlflow/server/prometheus_exporter.py#L1-L17)

## Troubleshooting Guide
Common issues when deploying MLflow tracking servers and their solutions:

1. **Database Migration Failures**: When running `mlflow db upgrade`, ensure you have a backup and sufficient time for large databases. For the `create_latest_metrics_table` migration, performance depends on the number of unique (run_id, metric_key) tuples.

2. **Authentication Issues**: Verify that the authentication app is properly installed (`pip install mlflow[auth]`) and that the `--app-name` parameter matches an available app.

3. **Artifact Storage Access**: Ensure proper credentials and permissions for cloud storage providers. For S3, configure AWS credentials via environment variables, IAM roles, or credential files.

4. **Network Latency**: For distributed environments, consider deploying the tracking server in the same region as the training workloads to minimize latency.

5. **Server Startup Issues**: Check that required dependencies are installed and that port 5000 (or the specified port) is not already in use.

**Section sources**
- [db.py](file://mlflow/db.py#L1-L28)
- [store/db_migrations/README.md](file://mlflow/store/db_migrations/README.md#L1-L61)
- [auth/__init__.py](file://mlflow/server/auth/__init__.py#L156-L162)
- [rest_utils.py](file://mlflow/utils/rest_utils.py#L627-L647)

## Conclusion
Deploying MLflow with remote tracking servers requires careful consideration of backend storage, artifact repositories, security, and performance characteristics. The system provides flexible configuration options through command-line arguments and environment variables, supporting various database backends and cloud storage providers. Authentication and authorization can be customized to meet organizational requirements, with built-in support for basic authentication and extensibility for custom implementations.

For enterprise deployments, consider implementing high availability through load balancing, monitoring server health with Prometheus metrics, and establishing backup and disaster recovery procedures for both the database and artifact storage. Proper configuration of connection pooling and server timeouts will ensure reliable operation under heavy loads.