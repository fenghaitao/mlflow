# Google Cloud Integration

<cite>
**Referenced Files in This Document**   
- [gcs_artifact_repo.py](file://mlflow/store/artifact/gcs_artifact_repo.py)
- [providers.py](file://mlflow/utils/providers.py)
- [google_adk.py](file://mlflow/tracing/otel/translation/google_adk.py)
- [providerUtils.ts](file://mlflow/server/js/src/gateway/utils/providerUtils.ts)
- [model_registry.proto](file://mlflow/protos/model_registry.proto)
- [model_registry_pb2.py](file://mlflow/protos/model_registry_pb2.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Google Cloud Storage Integration](#google-cloud-storage-integration)
3. [Vertex AI Deployment Configuration](#vertex-ai-deployment-configuration)
4. [Authentication and Service Account Management](#authentication-and-service-account-management)
5. [Model Registry and Deployment State](#model-registry-and-deployment-state)
6. [Regional Configuration and Endpoint Management](#regional-configuration-and-endpoint-management)
7. [Integration with MLflow Gateway](#integration-with-mlflow-gateway)
8. [Best Practices for Google Cloud Integration](#best-practices-for-google-cloud-integration)

## Introduction
MLflow provides comprehensive integration capabilities with Google Cloud Platform (GCP), enabling seamless model deployment to Vertex AI, artifact storage in Google Cloud Storage (GCS), and secure authentication through service accounts. This documentation details the implementation of MLflow's Google Cloud integration, focusing on Vertex AI deployment capabilities, GCS artifact repositories, and the configuration parameters that enable efficient model serving in GCP environments. The integration leverages MLflow's plugin architecture to support Google Cloud services while maintaining compatibility with the broader MLflow ecosystem.

## Google Cloud Storage Integration
MLflow integrates with Google Cloud Storage (GCS) through the `GCSArtifactRepository` class, which enables secure storage and retrieval of model artifacts. The integration supports both authenticated and anonymous access to GCS buckets, with configurable chunk sizes for upload and download operations.

The GCS artifact repository handles credential management through the `credential_refresh_def` parameter, which allows for dynamic credential refreshing during long-running operations. This is particularly important for large model deployments where credentials may expire during the upload process. The repository also supports multipart uploads for large files, ensuring reliable transfer of model artifacts.

```mermaid
classDiagram
class GCSArtifactRepository {
+_GCS_DOWNLOAD_CHUNK_SIZE int
+_GCS_UPLOAD_CHUNK_SIZE int
+_GCS_DEFAULT_TIMEOUT int
+credential_refresh_def function
+client GCS Client
+__init__(artifact_uri, client, credential_refresh_def)
+parse_gcs_uri(uri) tuple
+_get_bucket(bucket) Bucket
+_refresh_credentials() Bucket
+log_artifact(local_file, artifact_path)
+log_artifacts(local_dir, artifact_path)
+list_artifacts(path)
+_download_file(remote_file_path, local_path)
+delete_artifacts(artifact_path)
+create_multipart_upload(local_file, num_parts, artifact_path)
+complete_multipart_upload(local_file, upload_id, parts, artifact_path)
+abort_multipart_upload(local_file, upload_id, artifact_path)
}
class ArtifactRepository {
<<abstract>>
+artifact_uri string
+tracking_uri string
+registry_uri string
+__init__(artifact_uri, tracking_uri, registry_uri)
+log_artifact(local_file, artifact_path)
+log_artifacts(local_dir, artifact_path)
+list_artifacts(path)
+download_artifacts(path, dst_path)
+_download_file(remote_file_path, local_path)
+delete_artifacts(artifact_path)
}
class MultipartUploadMixin {
<<interface>>
+create_multipart_upload(local_file, num_parts, artifact_path)
+complete_multipart_upload(local_file, upload_id, parts, artifact_path)
+abort_multipart_upload(local_file, upload_id, artifact_path)
}
GCSArtifactRepository --|> ArtifactRepository
GCSArtifactRepository ..> MultipartUploadMixin
```

**Diagram sources**
- [gcs_artifact_repo.py](file://mlflow/store/artifact/gcs_artifact_repo.py#L36-L301)

**Section sources**
- [gcs_artifact_repo.py](file://mlflow/store/artifact/gcs_artifact_repo.py#L1-L301)

## Vertex AI Deployment Configuration
MLflow's integration with Vertex AI is configured through the deployment plugin system, which allows for target-specific configuration parameters. The Vertex AI deployment target supports service account-based authentication, project identification, and regional configuration.

The deployment configuration includes parameters for specifying the GCP project ID, region, and service account credentials. These parameters are defined in the providers configuration and are validated during deployment creation. The integration supports both direct model deployment and deployment through the MLflow Gateway, enabling flexible deployment patterns based on organizational requirements.

```mermaid
flowchart TD
A[MLflow Model] --> B{Deployment Target}
B --> C[Vertex AI]
C --> D[Configuration Parameters]
D --> E[vertex_credentials]
D --> F[vertex_project]
D --> G[vertex_location]
E --> H[Service Account JSON]
F --> I[GCP Project ID]
G --> J[Region e.g., us-central1]
H --> K[Authentication]
I --> L[Resource Isolation]
J --> M[Regional Availability]
K --> N[Secure Deployment]
L --> N
M --> N
N --> O[Vertex AI Endpoint]
```

**Diagram sources**
- [providers.py](file://mlflow/utils/providers.py#L212-L237)

**Section sources**
- [providers.py](file://mlflow/utils/providers.py#L212-L237)

## Authentication and Service Account Management
MLflow supports Google Cloud authentication through service account JSON credentials, which are securely managed within the deployment configuration. The authentication system is designed to handle credential refresh for long-running operations, ensuring that deployments can complete successfully even when credentials expire.

The service account configuration requires the JSON key file contents, which contain the private key and other authentication information. This approach provides fine-grained access control through IAM policies, allowing organizations to implement least-privilege security practices. The credentials are marked as secret in the configuration, ensuring they are not exposed in logs or configuration files.

```mermaid
sequenceDiagram
participant MLflow as MLflow Client
participant GCP as Google Cloud Platform
participant SA as Service Account
MLflow->>MLflow : Load service account JSON
MLflow->>GCP : Authenticate with credentials
GCP->>SA : Validate service account
SA-->>GCP : Authentication successful
GCP-->>MLflow : Access token
MLflow->>GCP : Deploy model to Vertex AI
GCP->>GCP : Create endpoint
GCP->>GCP : Deploy model
GCP-->>MLflow : Deployment successful
Note over MLflow,GCP : Secure authentication flow with service account
```

**Diagram sources**
- [providers.py](file://mlflow/utils/providers.py#L213-L222)

**Section sources**
- [providers.py](file://mlflow/utils/providers.py#L212-L237)

## Model Registry and Deployment State
MLflow's model registry integration with Vertex AI includes tracking of deployment job state, which provides visibility into the deployment process. The model version deployment job state is stored as part of the model version metadata, allowing users to monitor the status of deployments.

The deployment job state includes information about the current state of the deployment process, such as pending, running, or completed. This information is used to provide feedback during deployment operations and to support troubleshooting when deployments fail. The integration also supports connection state tracking, which indicates whether the deployment job is properly configured and accessible.

```mermaid
erDiagram
MODEL_VERSION ||--o{ DEPLOYMENT_JOB_STATE : has
MODEL_VERSION {
string name
int version
string status
string run_id
string source
string user_id
timestamp creation_timestamp
timestamp last_updated_timestamp
string description
json tags
json run_link
json aliases
json model_params
json model_metrics
}
DEPLOYMENT_JOB_STATE {
string state
string status_message
string job_id
string endpoint_name
timestamp start_time
timestamp end_time
json configuration
json error_details
}
DEPLOYMENT_JOB_CONNECTION ||--o{ DEPLOYMENT_JOB_STATE : has
DEPLOYMENT_JOB_CONNECTION {
string state
string connection_id
string owner
json parameters
timestamp created_at
timestamp updated_at
}
```

**Diagram sources**
- [model_registry.proto](file://mlflow/protos/model_registry.proto#L481-L482)
- [model_registry_pb2.py](file://mlflow/protos/model_registry_pb2.py#L347-L348)

**Section sources**
- [model_registry.proto](file://mlflow/protos/model_registry.proto#L456-L496)
- [model_registry_pb2.py](file://mlflow/protos/model_registry_pb2.py#L211-L360)

## Regional Configuration and Endpoint Management
MLflow's Vertex AI integration supports regional configuration, allowing deployments to be targeted to specific GCP regions. The default region is us-central1, but users can specify alternative regions based on their requirements for latency, data residency, or service availability.

The regional configuration is important for optimizing performance and complying with data governance requirements. Different regions may have different machine types available, which affects the cost and performance of deployed models. The integration also supports network peering configurations, enabling secure connectivity between Vertex AI endpoints and other resources in a VPC network.

```mermaid
graph TD
A[MLflow Deployment] --> B[Region Selection]
B --> C[us-central1]
B --> D[us-east4]
B --> E[europe-west4]
B --> F[asia-east1]
C --> G[Machine Types]
D --> G
E --> G
F --> G
G --> H[Cost Optimization]
G --> I[Performance]
G --> J[Availability]
H --> K[Preemptible Instances]
I --> L[Low Latency]
J --> M[High Availability]
K --> N[Cost-Effective Serving]
L --> O[Responsive Applications]
M --> P[Reliable Service]
```

**Diagram sources**
- [providers.py](file://mlflow/utils/providers.py#L230-L236)

**Section sources**
- [providers.py](file://mlflow/utils/providers.py#L212-L237)

## Integration with MLflow Gateway
MLflow Gateway provides an additional layer of integration with Vertex AI, enabling proxy-based access to deployed models. The gateway supports Vertex AI as a provider, allowing users to configure endpoints that route requests to Vertex AI models.

The gateway integration includes support for various Vertex AI model types, including text, chat, embedding, and vision models. Each model type is represented as a variant in the provider configuration, allowing for specialized configuration and routing. This enables organizations to deploy multiple model types through a unified gateway interface.

```mermaid
classDiagram
class VertexAIProvider {
+VERTEX_AI_VARIANT_NAMES map
+formatProviderName(provider) string
+getProviderConfig(provider) config
+validateConfiguration(config) boolean
+createEndpoint(config) endpoint
+deleteEndpoint(endpoint) void
+predict(endpoint, request) response
}
class GatewayProvider {
<<interface>>
+getProviderConfig(provider) config
+validateConfiguration(config) boolean
+createEndpoint(config) endpoint
+deleteEndpoint(endpoint) void
+predict(endpoint, request) response
}
class ProviderUtils {
+PROVIDER_DISPLAY_NAMES map
+VERTEX_AI_VARIANT_NAMES map
+formatProviderName(provider) string
+formatAuthMethodName(authMethod) string
}
VertexAIProvider --|> GatewayProvider
ProviderUtils ..> VertexAIProvider
```

**Diagram sources**
- [providerUtils.ts](file://mlflow/server/js/src/gateway/utils/providerUtils.ts#L86-L121)
- [google_adk.py](file://mlflow/tracing/otel/translation/google_adk.py#L13-L14)

**Section sources**
- [providerUtils.ts](file://mlflow/server/js/src/gateway/utils/providerUtils.ts#L83-L130)
- [google_adk.py](file://mlflow/tracing/otel/translation/google_adk.py#L1-L14)

## Best Practices for Google Cloud Integration
When integrating MLflow with Google Cloud, several best practices should be followed to ensure secure, reliable, and cost-effective deployments:

1. **Service Account Management**: Use dedicated service accounts with least-privilege permissions for MLflow deployments. Regularly rotate service account keys and monitor their usage.

2. **Regional Selection**: Choose regions based on data residency requirements, latency needs, and available machine types. Consider using regions with preemptible instances for cost optimization.

3. **Artifact Storage**: Use GCS for artifact storage with appropriate storage classes based on access patterns. Implement lifecycle policies to manage storage costs.

4. **Network Configuration**: Configure VPC peering or Private Service Connect for secure connectivity between MLflow and Vertex AI endpoints.

5. **Monitoring and Logging**: Enable Cloud Logging and Cloud Monitoring for deployed models to track performance, errors, and usage patterns.

6. **Cost Optimization**: Use appropriate machine types and consider preemptible instances for non-critical workloads. Monitor usage and set budget alerts.

7. **Security**: Enable IAM auditing and regularly review access permissions. Use VPC Service Controls to protect against data exfiltration.

8. **Deployment Strategies**: Implement canary deployments and blue-green deployment patterns to minimize risk during model updates.

These practices ensure that MLflow deployments on Google Cloud are secure, reliable, and cost-effective while maintaining compliance with organizational policies and regulatory requirements.

**Section sources**
- [providers.py](file://mlflow/utils/providers.py#L212-L237)
- [gcs_artifact_repo.py](file://mlflow/store/artifact/gcs_artifact_repo.py#L1-L301)
- [providerUtils.ts](file://mlflow/server/js/src/gateway/utils/providerUtils.ts#L83-L130)