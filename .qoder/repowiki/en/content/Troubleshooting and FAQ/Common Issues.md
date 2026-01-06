# Common Issues

<cite>
**Referenced Files in This Document**   
- [environment_variables.py](file://mlflow/environment_variables.py)
- [security.py](file://mlflow/server/security.py)
- [db.py](file://mlflow/db.py)
- [exceptions.py](file://mlflow/exceptions.py)
- [utils/server_cli_utils.py](file://mlflow/utils/server_cli_utils.py)
- [server/fastapi_app.py](file://mlflow/server/fastapi_app.py)
- [store/artifact/presigned_url_artifact_repo.py](file://mlflow/store/artifact/presigned_url_artifact_repo.py)
- [store/_unity_catalog/registry/uc_oss_rest_store.py](file://mlflow/store/_unity_catalog/registry/uc_oss_rest_store.py)
- [utils/_unity_catalog_utils.py](file://mlflow/utils/_unity_catalog_utils.py)
- [pyfunc/dbconnect_artifact_cache.py](file://mlflow/pyfunc/dbconnect_artifact_cache.py)
- [tests/server/test_security.py](file://tests/server/test_security.py)
- [tests/db/test_schema.py](file://tests/db/test_schema.py)
- [docs/docs/self-hosting/architecture/artifact-store.mdx](file://docs/docs/self-hosting/architecture/artifact-store.mdx)
- [docs/docs/classic-ml/tracking/tutorials/remote-server/index.mdx](file://docs/docs/classic-ml/tracking/tutorials/remote-server/index.mdx)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Tracking Server Configuration Issues](#tracking-server-configuration-issues)
3. [Artifact Repository Connectivity Problems](#artifact-repository-connectivity-problems)
4. [Model Registry Access Challenges](#model-registry-access-challenges)
5. [Deployment Failures](#deployment-failures)
6. [Configuration and Environment Variables](#configuration-and-environment-variables)
7. [Database Migration and Backend Store Issues](#database-migration-and-backend-store-issues)
8. [Cross-Origin Resource Sharing (CORS) Policies](#cross-origin-resource-sharing-cors-policies)
9. [Authentication and Permission Errors](#authentication-and-permission-errors)
10. [Troubleshooting Guide](#troubleshooting-guide)

## Introduction
This document addresses common issues encountered when using MLflow, focusing on tracking server configuration, artifact repository connectivity, model registry access, and deployment failures. The analysis covers implementation details, invocation relationships, and usage patterns across the MLflow ecosystem. Key areas include configuration options, parameters, error conditions, and relationships between components such as the tracking server, backend store, and artifact store. The document provides solutions for issues related to environment variables, storage backend configuration, and cross-origin resource sharing (CORS) policies, with concrete examples from the actual codebase.

**Section sources**
- [environment_variables.py](file://mlflow/environment_variables.py#L1-L1210)
- [security.py](file://mlflow/server/security.py#L1-L116)

## Tracking Server Configuration Issues
MLflow tracking server configuration issues often stem from improper environment variable settings, incorrect URI configurations, or misaligned component relationships. The tracking server relies on a clear separation between the backend store (for metadata) and artifact store (for large files). A common issue occurs when starting a tracking server in `--artifacts-only` mode while providing a `--backend_store_uri` value, which creates a configuration conflict.

The server validation logic explicitly prevents this configuration:
```python
def artifacts_only_config_validation(artifacts_only: bool, backend_store_uri: str) -> None:
    if artifacts_only and not _is_default_backend_store_uri(backend_store_uri):
        msg = (
            "You are starting a tracking server in `--artifacts-only` mode and have provided a "
            f"value for `--backend_store_uri`: '{backend_store_uri}'. A tracking server in "
            "`--artifacts-only` mode cannot have a value set for `--backend_store_uri` to "
            "properly proxy access to the artifact storage location."
        )
        raise click.UsageError(message=msg)
```

This validation ensures that when running in artifacts-only mode, the server properly proxies access to the artifact storage location without attempting to manage backend store operations. The tracking URI scheme is also critical for proper server operation, with different schemes determining the appropriate store implementation through the registry pattern.

**Section sources**
- [utils/server_cli_utils.py](file://mlflow/utils/server_cli_utils.py#L48-L56)
- [tracking/registry.py](file://mlflow/tracking/registry.py#L34-L68)

## Artifact Repository Connectivity Problems
Artifact repository connectivity issues typically involve authentication problems, network connectivity failures, or configuration mismatches between the client and server. MLflow supports various artifact storage locations including Amazon S3, Azure Blob Storage, Google Cloud Storage, SFTP server, and NFS, with the default being a local file system `./mlruns` directory.

The artifact store handles credential management through presigned URLs and header-based authentication:
```python
def _get_write_credential_infos(self, remote_file_paths):
    endpoint, method = FILESYSTEM_METHOD_TO_INFO[CreateUploadUrlRequest]
    credential_infos = []
    for relative_path in remote_file_paths:
        fs_full_path = posixpath.join(self.artifact_uri, relative_path)
        req_body = message_to_json(CreateUploadUrlRequest(path=fs_full_path))
        response_proto = CreateUploadUrlResponse()
        resp = call_endpoint(
            host_creds=self.db_creds,
            endpoint=endpoint,
            method=method,
            json_body=req_body,
            response_proto=response_proto,
        )
        headers = [
            ArtifactCredentialInfo.HttpHeader(name=header.name, value=header.value)
            for header in resp.headers
        ]
        credential_infos.append(ArtifactCredentialInfo(signed_uri=resp.url, headers=headers))
    return credential_infos
```

Common connectivity issues include:
- Missing or incorrect S3 endpoint URLs (`MLFLOW_S3_ENDPOINT_URL`)
- TLS certificate verification problems with S3 (`MLFLOW_S3_IGNORE_TLS`)
- Kerberos authentication failures for HDFS operations
- Proxy configuration issues for external artifact repositories

The system also supports multipart upload and download operations for large files, with configurable thresholds and chunk sizes through environment variables like `MLFLOW_MULTIPART_UPLOAD_MINIMUM_FILE_SIZE` and `MLFLOW_MULTIPART_UPLOAD_CHUNK_SIZE`.

**Section sources**
- [store/artifact/presigned_url_artifact_repo.py](file://mlflow/store/artifact/presigned_url_artifact_repo.py#L65-L84)
- [environment_variables.py](file://mlflow/environment_variables.py#L349-L363)
- [docs/docs/self-hosting/architecture/artifact-store.mdx](file://docs/docs/self-hosting/architecture/artifact-store.mdx#L1-L16)

## Model Registry Access Challenges
Model registry access issues often involve permission errors, API endpoint misconfigurations, or problems with Unity Catalog integration. The model registry operations are implemented through REST APIs with specific endpoints for creating, updating, and retrieving registered models.

For Unity Catalog OSS (Open Source Software) integration, the implementation includes:
```python
def create_registered_model(self, name, description=None, tags=None):
    [catalog_name, schema_name, model_name] = name.split(".")
    comment = description or ""
    req_body = message_to_json(
        CreateRegisteredModel(
            name=model_name,
            catalog_name=catalog_name,
            schema_name=schema_name,
            comment=comment,
        )
    )
    registered_model_info = self._call_endpoint(CreateRegisteredModel, req_body)
    return get_registered_model_from_uc_oss_proto(registered_model_info)
```

Common access challenges include:
- Incorrect model name format (must be in `catalog.schema.model` format)
- Permission denied errors when creating or updating models
- Issues with deployment job integration
- Problems with tag and alias management

The system converts between internal proto representations and Python entities:
```python
def registered_model_from_uc_proto(uc_proto: ProtoRegisteredModel) -> RegisteredModel:
    return RegisteredModel(
        name=uc_proto.name,
        creation_timestamp=uc_proto.creation_timestamp,
        last_updated_timestamp=uc_proto.last_updated_timestamp,
        description=uc_proto.description,
        aliases=[
            RegisteredModelAlias(alias=alias.alias, version=alias.version)
            for alias in (uc_proto.aliases or [])
        ],
        tags=[RegisteredModelTag(key=tag.key, value=tag.value) for tag in (uc_proto.tags or [])],
        deployment_job_id=uc_proto.deployment_job_id,
        deployment_job_state=RegisteredModelDeploymentJobState.to_string(
            uc_proto.deployment_job_state
        ),
    )
```

**Section sources**
- [store/_unity_catalog/registry/uc_oss_rest_store.py](file://mlflow/store/_unity_catalog/registry/uc_oss_rest_store.py#L114-L257)
- [utils/_unity_catalog_utils.py](file://mlflow/utils/_unity_catalog_utils.py#L122-L137)

## Deployment Failures
Deployment failures in MLflow can stem from various sources including configuration issues, resource limitations, and network problems. The system provides extensive timeout and retry configurations to handle transient failures during deployment operations.

Key deployment-related environment variables include:
- `MLFLOW_DEPLOYMENT_PREDICT_TIMEOUT`: Timeout for a single HTTP request to a deployment endpoint
- `MLFLOW_DEPLOYMENT_PREDICT_TOTAL_TIMEOUT`: Total time limit for all retry attempts combined
- `MLFLOW_DEPLOYMENT_CLIENT_HTTP_REQUEST_TIMEOUT`: Timeout for non-predict operations

Proxy configuration is also critical for deployment scenarios:
```python
if proxies_enabled:
    proxy_variables = {
        "http_proxy": "http://user:password@proxy.example.net:1234",
        "https_proxy": "https://user:password@proxy.example.net:1234",
        "no_proxy": "localhost",
    }
    for k, v in proxy_variables.items():
        monkeypatch.setenv(k, v)
```

Common deployment issues include:
- Timeout errors during model prediction
- Proxy authentication failures
- Resource exhaustion on the target deployment platform
- Incompatible model flavors for the target deployment environment
- Missing dependencies in the deployment environment

The system also supports asynchronous logging during deployment operations, which can be configured with `MLFLOW_ENABLE_ASYNC_LOGGING` and `MLFLOW_ASYNC_LOGGING_THREADPOOL_SIZE`.

**Section sources**
- [tests/sagemaker/test_sagemaker_deployment_client.py](file://tests/sagemaker/test_sagemaker_deployment_client.py#L552-L579)
- [environment_variables.py](file://mlflow/environment_variables.py#L572-L586)

## Configuration and Environment Variables
MLflow's behavior is heavily influenced by environment variables, which control various aspects of the system from tracking URI to security settings. The environment variables follow a consistent naming convention with public variables beginning with `MLFLOW_` and internal-use variables starting with `_MLFLOW_`.

Key configuration categories include:

### Tracking and Authentication
- `MLFLOW_TRACKING_URI`: Specifies the tracking URI
- `MLFLOW_TRACKING_USERNAME` and `MLFLOW_TRACKING_PASSWORD`: Credentials for tracking server authentication
- `MLFLOW_TRACKING_TOKEN`: Bearer token for authentication
- `MLFLOW_TRACKING_INSECURE_TLS`: Controls TLS verification

### Artifact Storage
- `MLFLOW_S3_ENDPOINT_URL`: S3 endpoint URL for artifact operations
- `MLFLOW_S3_IGNORE_TLS`: Skip TLS certificate verification for S3
- `MLFLOW_GCS_DOWNLOAD_CHUNK_SIZE`: Chunk size for GCS downloads
- `MLFLOW_ENABLE_MULTIPART_UPLOAD`: Enable multipart upload for large artifacts

### Database and Backend Store
- `MLFLOW_SQLALCHEMYSTORE_POOL_SIZE`: Connection pool size for SQLAlchemy
- `MLFLOW_SQLALCHEMYSTORE_POOL_RECYCLE`: Connection recycle time
- `MLFLOW_SQLALCHEMYSTORE_MAX_OVERFLOW`: Maximum overflow connections

### Security and CORS
- `MLFLOW_SERVER_CORS_ALLOWED_ORIGINS`: Allowed CORS origins
- `MLFLOW_SERVER_ALLOWED_HOSTS`: Allowed host headers
- `MLFLOW_SERVER_X_FRAME_OPTIONS`: X-Frame-Options header value

### Performance and Timeouts
- `MLFLOW_HTTP_REQUEST_MAX_RETRIES`: Maximum HTTP request retries
- `MLFLOW_HTTP_REQUEST_TIMEOUT`: HTTP request timeout
- `MLFLOW_SCORING_SERVER_REQUEST_TIMEOUT`: Model scoring server timeout

These variables provide fine-grained control over MLflow's behavior, allowing administrators to tune the system for their specific environment and requirements.

**Section sources**
- [environment_variables.py](file://mlflow/environment_variables.py#L1-L1210)

## Database Migration and Backend Store Issues
Database migration and backend store issues are critical concerns when upgrading MLflow or changing the underlying database technology. The system includes specific commands and validation checks to ensure database schema compatibility.

The database upgrade process is managed through the CLI:
```python
@commands.command()
@click.argument("url")
def upgrade(url):
    """
    Upgrade the schema of an MLflow tracking database to the latest supported version.
    
    **IMPORTANT**: Schema migrations can be slow and are not guaranteed to be transactional -
    **always take a backup of your database before running migrations**.
    """
    import mlflow.store.db.utils
    
    engine = mlflow.store.db.utils.create_sqlalchemy_engine_with_retry(url)
    mlflow.store.db.utils._upgrade_db(engine)
```

A critical validation test ensures schema consistency:
```python
def test_schema_is_up_to_date():
    initialize_database()
    tracking_uri = get_tracking_uri()
    schema_path = get_schema_path(tracking_uri)
    existing_schema = schema_path.read_text()
    latest_schema = dump_schema(tracking_uri)
    dialect = get_database_dialect(tracking_uri)
    update_command = get_schema_update_command(dialect)
    message = (
        f"{schema_path.relative_to(Path.cwd())} is not up-to-date. "
        f"Please run this command to update it: {update_command}"
    )
    diff = "".join(
        difflib.ndiff(
            existing_schema.splitlines(keepends=True), latest_schema.splitlines(keepends=True)
        )
    )
    assert schema_equal(existing_schema, latest_schema), message
```

Common database issues include:
- Schema version mismatches after MLflow upgrades
- Connection pool exhaustion under high load
- Long-running migrations that impact availability
- Permission errors when accessing the database
- Network connectivity problems between MLflow server and database

The system uses SQLAlchemy engine caching to prevent connection pool leaks when multiple store instances are created with the same database URI:
```python
_engine_map: dict[str, sqlalchemy.engine.Engine] = {}
_engine_map_lock = threading.Lock()

@classmethod
def _get_or_create_engine(cls, db_uri: str) -> sqlalchemy.engine.Engine:
    """Get a cached engine or create a new one for the given database URI."""
    if db_uri not in cls._engine_map:
        with cls._engine_map_lock:
            if db_uri not in cls._engine_map:
                cls._engine_map[db_uri] = create_sqlalchemy_engine_with_retry(db_uri)
    return cls._engine_map[db_uri]
```

**Section sources**
- [db.py](file://mlflow/db.py#L1-L28)
- [tests/db/test_schema.py](file://tests/db/test_schema.py#L128-L157)
- [store/model_registry/sqlalchemy_store.py](file://mlflow/store/model_registry/sqlalchemy_store.py#L100-L112)

## Cross-Origin Resource Sharing (CORS) Policies
Cross-Origin Resource Sharing (CORS) policies in MLflow are configured through environment variables and implemented using Flask-CORS middleware. The system provides fine-grained control over which origins and hosts are allowed to access the MLflow server.

CORS configuration is managed through environment variables:
```python
@pytest.mark.parametrize(
    ("env_var", "env_value", "expected_result"),
    [
        (
            "MLFLOW_SERVER_CORS_ALLOWED_ORIGINS",
            "http://app1.com,http://app2.com",
            ["http://app1.com", "http://app2.com"],
        ),
        ("MLFLOW_SERVER_ALLOWED_HOSTS", "app1.com,app2.com:8080", ["app1.com", "app2.com:8080"]),
    ],
)
def test_environment_variable_configuration(
    env_var, env_value, expected_result, monkeypatch: pytest.MonkeyPatch
):
    monkeypatch.setenv(env_var, env_value)
    if "ORIGINS" in env_var:
        result = security.get_allowed_origins()
        for expected in expected_result:
            assert expected in result
    else:
        result = security.get_allowed_hosts()
        for expected in expected_result:
            assert expected in result
```

The security middleware initialization handles CORS configuration:
```python
def init_security_middleware(app: Flask) -> None:
    """
    Initialize security middleware for Flask application.
    
    This configures:
    - Host header validation (DNS rebinding protection)
    - CORS protection via Flask-CORS
    - Security headers
    """
    if MLFLOW_SERVER_DISABLE_SECURITY_MIDDLEWARE.get() == "true":
        return

    allowed_origins = get_allowed_origins()
    allowed_hosts = get_allowed_hosts()
    x_frame_options = MLFLOW_SERVER_X_FRAME_OPTIONS.get()

    if allowed_origins and "*" in allowed_origins:
        CORS(app, resources={r"/*": {"origins": "*"}}, supports_credentials=True)
    else:
        cors_origins = (allowed_origins or []) + LOCALHOST_ORIGIN_PATTERNS
        CORS(
            app,
            resources={r"/*": {"origins": cors_origins}},
            supports_credentials=True,
            methods=["GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"],
        )
```

Common CORS issues include:
- Requests being blocked due to missing or incorrect origin headers
- Wildcard origins (`*`) not being properly configured with credentials
- Preflight OPTIONS requests not being handled correctly
- Host header validation blocking legitimate requests
- X-Frame-Options conflicts with embedding in iframes

The system also handles preflight requests by converting 200 responses to NO_CONTENT for API endpoints:
```python
@app.after_request
def add_security_headers(response: Response) -> Response:
    if (
        request.method == "OPTIONS"
        and response.status_code == 200
        and is_api_endpoint(request.path)
    ):
        response.status_code = HTTPStatus.NO_CONTENT
        response.data = b""
```

**Section sources**
- [server/security.py](file://mlflow/server/security.py#L28-L116)
- [tests/server/test_security.py](file://tests/server/test_security.py#L243-L265)

## Authentication and Permission Errors
Authentication and permission errors in MLflow can occur at multiple levels, including tracking server access, model registry operations, and artifact repository interactions. The system supports various authentication methods and provides detailed error handling for permission-related issues.

Key authentication environment variables include:
- `MLFLOW_TRACKING_USERNAME` and `MLFLOW_TRACKING_PASSWORD`: Basic authentication credentials
- `MLFLOW_TRACKING_TOKEN`: Bearer token for authentication
- `MLFLOW_TRACKING_AWS_SIGV4`: AWS Signature Version 4 signing
- `MLFLOW_TRACKING_AUTH`: Custom authentication provider

The system handles authentication through request headers:
```python
def call_endpoint(
    host_creds: rest_utils.MlflowHostCreds,
    endpoint,
    method,
    json_body=None,
    params=None,
    response_proto=None,
    retries=5,
    timeout=60,
):
    """
    Makes an HTTP request to the remote server and returns a proto message.
    """
    hostname = host_creds.host
    auth_str = None
    if host_creds.username and host_creds.password:
        auth_str = base64.standard_b64encode(
            ("%s:%s" % (host_creds.username, host_creds.password)).encode("utf-8")
        ).decode("utf-8")
        auth_str = "Basic %s" % auth_str
    elif host_creds.token:
        auth_str = "Bearer %s" % host_creds.token
```

Common authentication issues include:
- Incorrect username/password combinations
- Expired or invalid bearer tokens
- AWS SigV4 signature mismatches
- Missing authentication headers in proxied requests
- Certificate validation failures with `MLFLOW_TRACKING_INSECURE_TLS`

Permission errors often occur when users attempt operations they don't have rights to perform, such as:
- Creating models in a registry they don't have write access to
- Reading artifacts from protected storage locations
- Updating experiment metadata without proper permissions
- Deleting runs or models without appropriate privileges

The system uses a comprehensive exception hierarchy to communicate these errors:
```python
class MlflowException(Exception):
    """
    Generic exception thrown to surface failure information about external-facing operations.
    The error message associated with this exception may be exposed to clients in HTTP responses
    for debugging purposes. If the error text is sensitive, raise a generic `Exception` object
    instead.
    """
    
class RestException(MlflowException):
    """Exception thrown on non 200-level responses from the REST API"""
    
ERROR_CODE_TO_HTTP_STATUS = {
    ErrorCode.Name(PERMISSION_DENIED): 403,
    ErrorCode.Name(UNAUTHENTICATED): 401,
    ErrorCode.Name(CUSTOMER_UNAUTHORIZED): 401,
}
```

**Section sources**
- [exceptions.py](file://mlflow/exceptions.py#L1-L220)
- [environment_variables.py](file://mlflow/environment_variables.py#L367-L396)

## Troubleshooting Guide
This troubleshooting guide provides solutions for common MLflow issues across various components.

### Tracking Server Issues
**Problem**: Tracking server fails to start with artifacts-only mode and backend store URI
**Solution**: Remove the `--backend_store_uri` parameter when starting in `--artifacts-only` mode, as these configurations are mutually exclusive.

**Problem**: Slow database performance
**Solution**: Tune database connection pool settings using environment variables:
- `MLFLOW_SQLALCHEMYSTORE_POOL_SIZE`: Increase for high concurrency
- `MLFLOW_SQLALCHEMYSTORE_POOL_RECYCLE`: Set to prevent stale connections
- `MLFLOW_SQLALCHEMYSTORE_MAX_OVERFLOW`: Control maximum overflow connections

### Artifact Repository Issues
**Problem**: S3 artifact operations fail with SSL/TLS errors
**Solution**: Configure S3 settings:
```bash
export MLFLOW_S3_IGNORE_TLS=true
export MLFLOW_S3_ENDPOINT_URL=https://s3.region.amazonaws.com
```

**Problem**: Large artifact uploads are slow or fail
**Solution**: Enable and configure multipart upload:
```bash
export MLFLOW_ENABLE_MULTIPART_UPLOAD=true
export MLFLOW_MULTIPART_UPLOAD_MINIMUM_FILE_SIZE=524288000  # 500MB
export MLFLOW_MULTIPART_UPLOAD_CHUNK_SIZE=10485760  # 10MB
```

### Model Registry Issues
**Problem**: Cannot create registered model in Unity Catalog
**Solution**: Ensure the model name follows the `catalog.schema.model` format and that you have appropriate permissions in the Unity Catalog.

**Problem**: Model version creation fails
**Solution**: Check that the source run exists and is accessible, and that you have write permissions to the registered model.

### Deployment Issues
**Problem**: Deployment predict requests timeout
**Solution**: Increase timeout values:
```bash
export MLFLOW_DEPLOYMENT_PREDICT_TIMEOUT=300
export MLFLOW_DEPLOYMENT_PREDICT_TOTAL_TIMEOUT=600
```

**Problem**: Proxy authentication fails for external services
**Solution**: Configure proxy environment variables:
```bash
export http_proxy=http://user:password@proxy.example.net:1234
export https_proxy=https://user:password@proxy.example.net:1234
export no_proxy=localhost
```

### CORS and Security Issues
**Problem**: Web UI requests are blocked by CORS policy
**Solution**: Configure allowed origins:
```bash
export MLFLOW_SERVER_CORS_ALLOWED_ORIGINS=http://localhost:3000,http://myapp.com
```

**Problem**: Host header validation blocks legitimate requests
**Solution**: Add the host to allowed hosts:
```bash
export MLFLOW_SERVER_ALLOWED_HOSTS=localhost,myserver.com
```

### General Troubleshooting Tips
1. Enable verbose logging with `MLFLOW_SQLALCHEMYSTORE_ECHO=true` for database issues
2. Check network connectivity between MLflow server and external services
3. Verify that all required dependencies are installed in the execution environment
4. Use `mlflow db upgrade` to ensure database schema is up-to-date after MLflow upgrades
5. Validate artifact repository credentials and permissions
6. Check file system permissions for local artifact storage
7. Monitor system resources (memory, disk space) during operations

**Section sources**
- [environment_variables.py](file://mlflow/environment_variables.py#L1-L1210)
- [security.py](file://mlflow/server/security.py#L1-L116)
- [db.py](file://mlflow/db.py#L1-L28)
- [exceptions.py](file://mlflow/exceptions.py#L1-L220)