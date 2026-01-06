# Plugin Development

<cite>
**Referenced Files in This Document**   
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py)
- [interface.py](file://mlflow/deployments/interface.py)
- [base.py](file://mlflow/deployments/base.py)
- [plugins.py](file://mlflow/utils/plugins.py)
- [registry.py](file://mlflow/tracking/request_auth/registry.py)
- [fake_deployment_plugin.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/fake_deployment_plugin.py)
- [request_auth_provider.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/request_auth_provider.py)
- [sqlalchemy_store.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/sqlalchemy_store.py)
- [providers.py](file://examples/gateway/plugin/my-llm/my_llm/providers.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Plugin Architecture Overview](#plugin-architecture-overview)
3. [Deployment Plugins](#deployment-plugins)
4. [Authentication Plugins](#authentication-plugins)
5. [Model Flavor Plugins](#model-flavor-plugins)
6. [Plugin Registration Mechanism](#plugin-registration-mechanism)
7. [Plugin Discovery Process](#plugin-discovery-process)
8. [Implementation Example: Custom Deployment Plugin](#implementation-example-custom-deployment-plugin)
9. [Relationships with Core Components](#relationships-with-core-components)
10. [Common Issues and Debugging](#common-issues-and-debugging)
11. [Testing and Integration](#testing-and-integration)
12. [Best Practices](#best-practices)

## Introduction
MLflow's plugin system enables extensibility through Python entry points, allowing developers to extend MLflow's functionality for various use cases including model deployment, authentication, and model flavor support. This document provides comprehensive guidance on developing MLflow plugins, covering implementation details, registration mechanisms, and integration patterns. The plugin architecture is designed to be modular and extensible, supporting multiple plugin types that can be discovered and loaded at runtime.

## Plugin Architecture Overview

```mermaid
graph TD
PluginSystem[MLflow Plugin System] --> DeploymentPlugins[Deployment Plugins]
PluginSystem --> AuthPlugins[Authentication Plugins]
PluginSystem --> ModelFlavorPlugins[Model Flavor Plugins]
PluginSystem --> GatewayPlugins[Gateway Plugins]
DeploymentPlugins --> PluginManager[PluginManager]
AuthPlugins --> RequestAuthProviderRegistry[RequestAuthProviderRegistry]
ModelFlavorPlugins --> FlavorBackendRegistry[FlavorBackendRegistry]
GatewayPlugins --> ProviderRegistry[ProviderRegistry]
PluginManager --> EntryPoint[Python Entry Points]
RequestAuthProviderRegistry --> EntryPoint
FlavorBackendRegistry --> EntryPoint
ProviderRegistry --> EntryPoint
EntryPoint --> SetupPy[setup.py/pyproject.toml]
SetupPy --> PackageDistribution[Package Distribution]
```

**Diagram sources**
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py)
- [registry.py](file://mlflow/tracking/request_auth/registry.py)

**Section sources**
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py#L1-L144)
- [registry.py](file://mlflow/tracking/request_auth/registry.py#L1-L61)

## Deployment Plugins

### Core Interface
Deployment plugins in MLflow implement a standardized interface through the `BaseDeploymentClient` class, which defines the contract for model deployment operations. The interface includes methods for creating, updating, deleting, and listing deployments, as well as prediction and explanation capabilities.

```mermaid
classDiagram
class BaseDeploymentClient {
+__init__(target_uri)
+create_deployment(name, model_uri, flavor, config, endpoint)
+update_deployment(name, model_uri, flavor, config, endpoint)
+delete_deployment(name, config, endpoint)
+list_deployments(endpoint)
+get_deployment(name, endpoint)
+predict(deployment_name, inputs, endpoint)
+predict_stream(deployment_name, inputs, endpoint)
+explain(deployment_name, df, endpoint)
+create_endpoint(name, config)
+update_endpoint(endpoint, config)
+delete_endpoint(endpoint)
+list_endpoints()
+get_endpoint(endpoint)
}
class PluginDeploymentClient {
+create_deployment(name, model_uri, flavor, config, endpoint)
+delete_deployment(name, config, endpoint)
+update_deployment(name, model_uri, flavor, config, endpoint)
+list_deployments(endpoint)
+get_deployment(name, endpoint)
+predict(deployment_name, inputs, endpoint)
+explain(deployment_name, df, endpoint)
+create_endpoint(name, config)
+update_endpoint(endpoint, config)
+delete_endpoint(endpoint)
+list_endpoints()
+get_endpoint(endpoint)
}
BaseDeploymentClient <|-- PluginDeploymentClient
PluginDeploymentClient : Implements required methods
```

**Diagram sources**
- [base.py](file://mlflow/deployments/base.py#L1-L359)
- [fake_deployment_plugin.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/fake_deployment_plugin.py#L1-L64)

**Section sources**
- [base.py](file://mlflow/deployments/base.py#L1-L359)
- [interface.py](file://mlflow/deployments/interface.py#L1-L103)

### Plugin Manager Implementation
The `DeploymentPlugins` class manages the lifecycle of deployment plugins, handling registration and discovery through Python entry points. It maintains a registry of available plugins and validates their implementation against the required interface.

```mermaid
sequenceDiagram
participant Client as "MLflow Client"
participant Interface as "deployments.interface"
participant PluginManager as "DeploymentPlugins"
participant EntryPoint as "Python Entry Points"
Client->>Interface : get_deploy_client(target_uri)
Interface->>PluginManager : plugin_store[target]
PluginManager->>EntryPoint : get_entry_points("mlflow.deployments")
EntryPoint-->>PluginManager : List of entry points
PluginManager->>PluginManager : Register entry points
PluginManager-->>Interface : Plugin object
Interface->>Interface : Find BaseDeploymentClient subclass
Interface-->>Client : Deployment client instance
```

**Diagram sources**
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py#L1-L144)
- [interface.py](file://mlflow/deployments/interface.py#L1-L103)

## Authentication Plugins

### Request Authentication Provider
Authentication plugins in MLflow implement the `RequestAuthProvider` interface, allowing for custom authentication mechanisms to be integrated with MLflow's tracking server. These plugins are discovered through the `mlflow.request_auth_provider` entry point.

```mermaid
classDiagram
class RequestAuthProvider {
<<abstract>>
+get_name() string
+get_auth() dict
}
class PluginRequestAuthProvider {
+get_name() string
+get_auth() dict
}
RequestAuthProvider <|-- PluginRequestAuthProvider
PluginRequestAuthProvider : Implements authentication logic
```

**Diagram sources**
- [request_auth_provider.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/request_auth_provider.py#L1-L12)
- [registry.py](file://mlflow/tracking/request_auth/registry.py#L1-L61)

**Section sources**
- [registry.py](file://mlflow/tracking/request_auth/registry.py#L1-L61)
- [request_auth_provider.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/request_auth_provider.py#L1-L12)

### Authentication Registry
The `RequestAuthProviderRegistry` manages the discovery and registration of authentication plugins, allowing MLflow to dynamically load authentication providers based on configuration.

```mermaid
flowchart TD
Start([Application Start]) --> RegisterEntryPoints["Register Entry Points"]
RegisterEntryPoints --> DiscoverPlugins["Discover Plugins via mlflow.request_auth_provider"]
DiscoverPlugins --> LoadPlugin["Load Plugin Class"]
LoadPlugin --> CreateInstance["Create Plugin Instance"]
CreateInstance --> AddToRegistry["Add to _request_auth_provider_registry"]
AddToRegistry --> End([Registry Ready])
style Start fill:#f9f,stroke:#333
style End fill:#bbf,stroke:#333
```

**Diagram sources**
- [registry.py](file://mlflow/tracking/request_auth/registry.py#L1-L61)

## Model Flavor Plugins

### Flavor Backend Registry
Model flavor plugins extend MLflow's model persistence capabilities by implementing custom serialization and deserialization logic for specific model types. These plugins are registered through the `mlflow.model_flavor` entry point and managed by the `FlavorBackendRegistry`.

```mermaid
classDiagram
class FlavorBackend {
<<abstract>>
+save_model(model, path, **kwargs)
+load_model(path, **kwargs)
+can_score_model_on_current_platform()
+can_estimate_model_size_on_current_platform()
}
class CustomFlavorBackend {
+save_model(model, path, **kwargs)
+load_model(path, **kwargs)
+can_score_model_on_current_platform()
+can_estimate_model_size_on_current_platform()
}
FlavorBackend <|-- CustomFlavorBackend
CustomFlavorBackend : Implements flavor-specific logic
```

**Section sources**
- [sqlalchemy_store.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/sqlalchemy_store.py#L1-L10)

## Plugin Registration Mechanism

### Entry Point Configuration
Plugins are registered through Python entry points in either `setup.py` or `pyproject.toml`. The entry point configuration specifies the plugin name, module, and entry point group.

```mermaid
flowchart TD
A[Plugin Package] --> B[setup.py or pyproject.toml]
B --> C["entry_points={\n 'mlflow.deployments': [\n 'my-deployment=my_plugin.deployment:PluginDeploymentClient'\n ],\n 'mlflow.request_auth_provider': [\n 'my-auth=my_plugin.auth:PluginRequestAuthProvider'\n ]\n}"]
C --> D[Package Installation]
D --> E[Entry Points Registered in Distribution]
E --> F[MLflow Discovers Plugins at Runtime]
style A fill:#f9f,stroke:#333
style F fill:#bbf,stroke:#333
```

**Section sources**
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py#L70-L77)
- [registry.py](file://mlflow/tracking/request_auth/registry.py#L15-L26)

## Plugin Discovery Process

### Dynamic Plugin Loading
MLflow discovers and loads plugins at runtime using Python's `importlib.metadata.entry_points()` function. The discovery process occurs when the plugin manager is initialized, ensuring that all available plugins are registered before they are needed.

```mermaid
sequenceDiagram
participant App as "Application"
participant PluginManager as "PluginManager"
participant EntryPoints as "importlib.metadata.entry_points()"
App->>PluginManager : Initialize
PluginManager->>PluginManager : __init__()
PluginManager->>PluginManager : register_entrypoints()
PluginManager->>EntryPoints : get_entry_points(group_name)
EntryPoints-->>PluginManager : List of EntryPoint objects
PluginManager->>PluginManager : Register each entry point
PluginManager-->>App : Ready for use
```

**Diagram sources**
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py#L70-L77)
- [plugins.py](file://mlflow/utils/plugins.py#L1-L10)

**Section sources**
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py#L70-L77)
- [plugins.py](file://mlflow/utils/plugins.py#L1-L10)

## Implementation Example: Custom Deployment Plugin

### Plugin Structure
A custom deployment plugin requires implementing the `BaseDeploymentClient` interface, defining the `run_local` and `target_help` functions, and registering the plugin through entry points.

```mermaid
flowchart TD
A[Custom Deployment Plugin] --> B[PluginDeploymentClient]
A --> C[run_local function]
A --> D[target_help function]
B --> E[Implement create_deployment]
B --> F[Implement delete_deployment]
B --> G[Implement update_deployment]
B --> H[Implement list_deployments]
B --> I[Implement get_deployment]
B --> J[Implement predict]
B --> K[Implement explain]
B --> L[Implement endpoint methods]
style A fill:#f9f,stroke:#333
```

**Section sources**
- [fake_deployment_plugin.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/fake_deployment_plugin.py#L1-L64)

### Required Interface Methods
The plugin must implement all abstract methods from `BaseDeploymentClient` and provide the required top-level functions.

```mermaid
classDiagram
class RequiredMethods {
<<Interface>>
+create_deployment(name, model_uri, flavor, config, endpoint)
+update_deployment(name, model_uri, flavor, config, endpoint)
+delete_deployment(name, config, endpoint)
+list_deployments(endpoint)
+get_deployment(name, endpoint)
+predict(deployment_name, inputs, endpoint)
+explain(deployment_name, df, endpoint)
+create_endpoint(name, config)
+update_endpoint(endpoint, config)
+delete_endpoint(endpoint)
+list_endpoints()
+get_endpoint(endpoint)
}
class Implementation {
+create_deployment(name, model_uri, flavor, config, endpoint)
+update_deployment(name, model_uri, flavor, config, endpoint)
+delete_deployment(name, config, endpoint)
+list_deployments(endpoint)
+get_deployment(name, endpoint)
+predict(deployment_name, inputs, endpoint)
+explain(deployment_name, df, endpoint)
+create_endpoint(name, config)
+update_endpoint(endpoint, config)
+delete_endpoint(endpoint)
+list_endpoints()
+get_endpoint(endpoint)
}
RequiredMethods <|-- Implementation
Implementation : Concrete implementation
```

**Diagram sources**
- [base.py](file://mlflow/deployments/base.py#L93-L359)
- [fake_deployment_plugin.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/fake_deployment_plugin.py#L1-L64)

## Relationships with Core Components

### Integration with Deployments API
Deployment plugins integrate with MLflow's deployments API, enabling model deployment operations through the `get_deploy_client` function.

```mermaid
sequenceDiagram
participant User as "User"
participant CLI as "MLflow CLI"
participant Interface as "deployments.interface"
participant Plugin as "Deployment Plugin"
User->>CLI : mlflow deployments create -t my-target ...
CLI->>Interface : get_deploy_client("my-target")
Interface->>Plugin : Create PluginDeploymentClient instance
Plugin->>Plugin : create_deployment(...)
Plugin-->>CLI : Deployment details
CLI-->>User : Success message
```

**Section sources**
- [interface.py](file://mlflow/deployments/interface.py#L15-L65)

### Model Registry Integration
Plugins can extend the model registry functionality by implementing custom storage backends or adding new metadata fields.

```mermaid
classDiagram
class ModelRegistry {
+create_registered_model()
+create_model_version()
+transition_model_version_stage()
+search_registered_models()
}
class CustomRegistryStore {
+create_registered_model()
+create_model_version()
+transition_model_version_stage()
+search_registered_models()
}
ModelRegistry <|-- CustomRegistryStore
CustomRegistryStore : Extends default behavior
```

**Section sources**
- [sqlalchemy_store.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/sqlalchemy_store.py#L1-L10)

## Common Issues and Debugging

### Plugin Loading Failures
Common issues with plugin loading include missing entry points, incorrect module paths, and dependency conflicts.

```mermaid
flowchart TD
A[Plugin Loading Issue] --> B{Check Entry Point}
B --> |Missing| C[Verify setup.py/pyproject.toml]
B --> |Incorrect| D[Check module path and class name]
A --> E{Check Dependencies}
E --> |Missing| F[Install required dependencies]
E --> |Version Conflict| G[Resolve version compatibility]
A --> H{Check Implementation}
H --> |Missing Methods| I[Implement all required methods]
H --> |Incorrect Signature| J[Match method signatures exactly]
style A fill:#f9f,stroke:#333
style C fill:#f96,stroke:#333
style D fill:#f96,stroke:#333
style F fill:#f96,stroke:#333
style G fill:#f96,stroke:#333
style I fill:#f96,stroke:#333
style J fill:#f96,stroke:#333
```

**Section sources**
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py#L101-L143)
- [registry.py](file://mlflow/tracking/request_auth/registry.py#L15-L26)

### Version Compatibility
Plugin compatibility issues often arise from version mismatches between MLflow and plugin dependencies.

```mermaid
flowchart LR
A[MLflow Version] --> B[Plugin Compatibility]
C[Plugin Dependencies] --> B
D[Python Version] --> B
E[Target Platform] --> B
B --> F{Compatible?}
F --> |Yes| G[Plugin Works]
F --> |No| H[Resolve Compatibility Issues]
style G fill:#9f9,stroke:#333
style H fill:#f96,stroke:#333
```

## Testing and Integration

### Unit Testing Plugins
Effective testing of MLflow plugins requires testing both the plugin interface and the integration with MLflow's plugin system.

```mermaid
flowchart TD
A[Unit Test Plugin] --> B[Test Interface Methods]
A --> C[Test Entry Point Registration]
A --> D[Test Error Handling]
A --> E[Test Configuration]
B --> F[Test create_deployment]
B --> G[Test update_deployment]
B --> H[Test delete_deployment]
B --> I[Test predict]
C --> J[Test plugin appears in entry points]
C --> K[Test plugin loads correctly]
D --> L[Test invalid inputs]
D --> M[Test network failures]
D --> N[Test authentication errors]
style A fill:#f9f,stroke:#333
```

**Section sources**
- [fake_deployment_plugin.py](file://tests/resources/mlflow-test-plugin/mlflow_test_plugin/fake_deployment_plugin.py#L1-L64)

## Best Practices

### Plugin Development Guidelines
Following best practices ensures that plugins are robust, maintainable, and compatible with MLflow's ecosystem.

```mermaid
flowchart TD
A[Best Practices] --> B[Follow Interface Contracts]
A --> C[Handle Errors Gracefully]
A --> D[Document Configuration Options]
A --> E[Support Asynchronous Operations]
A --> F[Implement Proper Logging]
A --> G[Test Thoroughly]
A --> H[Follow Versioning Semantics]
B --> I[Implement all required methods]
B --> J[Match method signatures exactly]
C --> K[Raise MlflowException for errors]
C --> L[Provide meaningful error messages]
D --> M[Document all config parameters]
D --> N[Provide examples]
style A fill:#f9f,stroke:#333
```

**Section sources**
- [base.py](file://mlflow/deployments/base.py)
- [plugin_manager.py](file://mlflow/deployments/plugin_manager.py)
- [registry.py](file://mlflow/tracking/request_auth/registry.py)