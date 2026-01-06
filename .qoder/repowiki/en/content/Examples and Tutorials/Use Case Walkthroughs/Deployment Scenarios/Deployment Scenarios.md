# Deployment Scenarios

<cite>
**Referenced Files in This Document**   
- [MLproject](file://examples/docker/MLproject)
- [Dockerfile](file://examples/docker/Dockerfile)
- [train.py](file://examples/docker/train.py)
- [ray_serve/README.md](file://examples/ray_serve/README.md)
- [ray_serve/train_model.py](file://examples/ray_serve/train_model.py)
- [databricks/dbconnect.py](file://examples/databricks/dbconnect.py)
- [spark_udf/spark_udf.py](file://examples/spark_udf/spark_udf.py)
- [spark_udf/spark_udf_with_prebuilt_env.py](file://examples/spark_udf/spark_udf_with_prebuilt_env.py)
- [virtualenv/project/MLproject](file://examples/virtualenv/project/MLproject)
- [virtualenv/project/entrypoint.py](file://examples/virtualenv/project/entrypoint.py)
- [virtualenv/project/python_env.yaml](file://examples/virtualenv/project/python_env.yaml)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Docker Container Deployment](#docker-container-deployment)
3. [Ray Serve Deployment](#ray-serve-deployment)
4. [Amazon SageMaker Deployment](#amazon-sagemaker-deployment)
5. [Databricks Deployment](#databricks-deployment)
6. [Spark UDF Deployment](#spark-udf-deployment)
7. [Virtualenv Deployment](#virtualenv-deployment)
8. [Deployment Target Selection Guide](#deployment-target-selection-guide)
9. [Common Deployment Issues](#common-deployment-issues)
10. [Conclusion](#conclusion)

## Introduction
MLflow provides multiple deployment options for machine learning models, enabling flexibility across different infrastructure requirements and operational constraints. This document details the implementation specifics for deploying models to various targets including Docker containers, Ray Serve, Amazon SageMaker, Databricks, Spark UDFs, and virtualenv environments. Each deployment scenario has unique configuration requirements, deployment commands, and runtime considerations that must be understood to ensure successful model serving.

**Section sources**
- [MLproject](file://examples/docker/MLproject)
- [Dockerfile](file://examples/docker/Dockerfile)

## Docker Container Deployment

MLflow supports Docker-based deployment through the MLproject specification and Dockerfile configuration. The deployment process involves packaging the model with its dependencies in a Docker container that can be deployed to various container orchestration platforms.

The MLproject file defines the Docker environment by specifying the container image name:
```yaml
name: docker-example
docker_env:
  image: mlflow-docker-example
```

The corresponding Dockerfile installs required dependencies and copies the training script and data:
```dockerfile
FROM python:3.8
RUN pip install mlflow azure-storage-blob numpy scipy pandas scikit-learn cloudpickle
COPY train.py .
COPY wine-quality.csv .
```

Deployment is initiated using MLflow's projects API, which builds the Docker image and runs the specified entry point command. The train.py script demonstrates model training with ElasticNet regression and logging to MLflow, including parameters, metrics, and the trained model.

**Section sources**
- [MLproject](file://examples/docker/MLproject#L1-L12)
- [Dockerfile](file://examples/docker/Dockerfile#L1-L7)
- [train.py](file://examples/docker/train.py#L1-L71)

## Ray Serve Deployment

Ray Serve integration allows for scalable model deployment with dynamic scaling capabilities. The deployment process involves training a model, registering it in the MLflow Model Registry, and deploying it via the Ray Serve plugin.

The deployment workflow consists of:
1. Training and registering a model in the MLflow Model Registry
2. Starting a Ray cluster and Ray Serve instance
3. Creating a deployment using the MLflow deployments API
4. Scaling the deployment by updating replica count
5. Making predictions via the deployment endpoint

The ray_serve example demonstrates deploying a GradientBoostingClassifier model trained on the Iris dataset. Deployment is created using:
```bash
mlflow deployments create -t ray-serve -m models:/RayMLflowIntegration/1 --name iris:v1
```

The deployment can be scaled to multiple replicas:
```bash
mlflow deployments update -t ray-serve --name iris:v1 --config num_replicas=2
```

Predictions are made by sending input data to the deployed model:
```bash
mlflow deployments predict -t ray-serve --name iris:v1 --input-path input.json
```

**Section sources**
- [ray_serve/README.md](file://examples/ray_serve/README.md#L1-L61)
- [ray_serve/train_model.py](file://examples/ray_serve/train_model.py#L1-L42)

## Amazon SageMaker Deployment

Although specific SageMaker deployment files are not present in the immediate examples, MLflow provides native integration with Amazon SageMaker for model deployment. The sagemaker module in MLflow (mlflow/sagemaker/) contains the necessary components for deploying models to SageMaker endpoints.

The deployment process typically involves:
1. Preparing the model for SageMaker deployment
2. Creating a SageMaker model from the MLflow model
3. Deploying the model to a SageMaker endpoint
4. Configuring auto-scaling and monitoring

MLflow's SageMaker integration handles the creation of Docker containers with the appropriate inference code and dependencies, and manages the deployment lifecycle through the SageMaker API.

**Section sources**
- [sagemaker/__init__.py](file://mlflow/sagemaker/__init__.py)
- [sagemaker/cli.py](file://mlflow/sagemaker/cli.py)

## Databricks Deployment

MLflow provides seamless integration with Databricks for model deployment and inference. The dbconnect.py example demonstrates how to use Databricks Connect to train a model locally and deploy it for inference on a Databricks cluster.

The deployment workflow includes:
1. Establishing a connection to a Databricks cluster using DatabricksSession
2. Training a model locally
3. Logging the model to the Databricks MLflow tracking server
4. Creating a Spark UDF from the logged model for distributed inference

Key aspects of Databricks deployment:
- Uses Databricks Connect for remote Spark session creation
- Leverages MLflow's pyfunc.spark_udf for creating distributed inference functions
- Supports environment management with env_manager parameter
- Enables batch inference on large datasets using Spark

```python
spark = DatabricksSession.builder.remote(
    host=wc.config.host,
    token=wc.config.token,
    cluster_id=args.cluster_id,
).getOrCreate()

pyfunc_udf = mlflow.pyfunc.spark_udf(
    spark,
    model_info.model_uri,
    env_manager="local",
    result_type=DoubleType(),
)
```

**Section sources**
- [databricks/dbconnect.py](file://examples/databricks/dbconnect.py#L1-L57)

## Spark UDF Deployment

MLflow's Spark UDF integration enables model deployment for batch inference on Spark clusters. This approach leverages Spark's distributed computing capabilities for scalable model inference.

The spark_udf.py example demonstrates creating a Spark UDF from a trained scikit-learn model:
```python
pyfunc_udf = mlflow.pyfunc.spark_udf(spark, model_info.model_uri, env_manager="conda")
result = infer_spark_df.select(pyfunc_udf(*X.columns).alias("predictions")).toPandas()
```

Key features of Spark UDF deployment:
- **Environment reproducibility**: The env_manager parameter ensures the exact dependency versions used during training are reproduced
- **Distributed inference**: Leverages Spark's parallel processing capabilities
- **Type safety**: Supports specifying result types for type-safe operations
- **Prebuilt environments**: For improved performance, prebuilt environment archives can be used to avoid environment reconstruction

The spark_udf_with_prebuilt_env.py example shows advanced usage with prebuilt environments:
```python
pyfunc_udf = mlflow.pyfunc.spark_udf(spark, model_uri, prebuilt_env_uri=model_env_uc_path)
```

This approach is particularly valuable in Databricks environments where model environments can be prebuilt and stored in Unity Catalog volumes.

**Section sources**
- [spark_udf/spark_udf.py](file://examples/spark_udf/spark_udf.py#L1-L24)
- [spark_udf/spark_udf_with_prebuilt_env.py](file://examples/spark_udf/spark_udf_with_prebuilt_env.py#L1-L45)

## Virtualenv Deployment

Virtualenv deployment allows for isolated Python environment creation with specific dependency versions. The virtualenv example demonstrates how MLflow can create reproducible environments for model deployment.

The MLproject configuration specifies the Python environment file:
```yaml
name: virtualenv-example
python_env: python_env.yaml
```

The python_env.yaml file defines dependencies:
```yaml
dependencies:
  - -r requirements.txt
```

The entrypoint.py script verifies the virtual environment and dependency versions:
```python
if args.test:
    assert "VIRTUAL_ENV" in os.environ
    assert sys.version_info[:3] == (3, 8, 18), sys.version_info
    assert sklearn.__version__ == "1.0.2", sklearn.__version__
```

This deployment approach ensures:
- Python version consistency
- Package version reproducibility
- Isolated execution environment
- Reliable model behavior across different systems

**Section sources**
- [virtualenv/project/MLproject](file://examples/virtualenv/project/MLproject#L1-L8)
- [virtualenv/project/entrypoint.py](file://examples/virtualenv/project/entrypoint.py#L1-L34)
- [virtualenv/project/python_env.yaml](file://examples/virtualenv/project/python_env.yaml#L1-L3)

## Deployment Target Selection Guide

Selecting the appropriate deployment target depends on several factors including scalability requirements, infrastructure constraints, and operational complexity.

**Docker containers** are ideal for:
- Deployment to container orchestration platforms (Kubernetes, ECS)
- Environments requiring strict dependency isolation
- Hybrid or multi-cloud deployments
- When fine-grained control over the runtime environment is needed

**Ray Serve** is best suited for:
- Applications requiring dynamic scaling
- Real-time inference with low latency requirements
- Complex model serving patterns (ensemble models, canary deployments)
- When horizontal scaling across multiple nodes is required

**Amazon SageMaker** is recommended for:
- AWS-centric infrastructure
- Enterprise-grade model serving with built-in monitoring
- Integration with SageMaker Pipelines and Feature Store
- When managed infrastructure with auto-scaling is preferred

**Databricks** deployment works well for:
- Organizations already using Databricks for data engineering
- Batch inference on large datasets
- Integration with Delta Lake and Unity Catalog
- When leveraging existing Databricks compute resources

**Spark UDFs** are optimal for:
- Batch processing workloads
- Integration with existing Spark pipelines
- Large-scale data transformation and inference
- When leveraging Spark's distributed computing capabilities

**Virtualenv** deployment is appropriate for:
- Development and testing environments
- Simple deployment scenarios
- When lightweight environment isolation is sufficient
- Local execution and debugging

## Common Deployment Issues

Several common issues can arise during MLflow model deployment:

**Environment conflicts**: Dependency version mismatches between training and deployment environments can cause failures. This is mitigated by using environment managers (conda, virtualenv) and specifying exact dependency versions.

**Serialization problems**: Models may fail to serialize or deserialize correctly. MLflow's model flavor system helps ensure compatibility, but custom models may require special handling.

**Performance bottlenecks**: 
- Cold start latency in serverless deployments
- Memory constraints in containerized environments
- Network latency in distributed inference
- CPU/GPU utilization imbalances

**Configuration issues**:
- Incorrect model URIs
- Missing dependency specifications
- Improper entry point definitions
- Environment variable misconfigurations

**Security considerations**:
- Proper authentication and authorization
- Secure model artifact storage
- Protection against model extraction attacks
- Input validation to prevent injection attacks

## Conclusion

MLflow provides a comprehensive set of deployment options for machine learning models, each with specific strengths and use cases. Understanding the implementation details, configuration requirements, and operational characteristics of each deployment target is essential for successful model serving. By leveraging MLflow's standardized model format and deployment APIs, organizations can achieve consistent, reproducible, and scalable model deployment across diverse infrastructure environments.

The choice of deployment target should be guided by scalability requirements, infrastructure constraints, operational complexity, and integration needs. Proper attention to environment management, dependency specification, and performance optimization ensures reliable model serving in production environments.