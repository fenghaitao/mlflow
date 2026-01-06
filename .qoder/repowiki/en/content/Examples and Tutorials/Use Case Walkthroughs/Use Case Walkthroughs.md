# Use Case Walkthroughs

<cite>
**Referenced Files in This Document**   
- [multistep_workflow/MLproject](file://examples/multistep_workflow/MLproject)
- [multistep_workflow/main.py](file://examples/multistep_workflow/main.py)
- [hyperparam/MLproject](file://examples/hyperparam/MLproject)
- [hyperparam/search_hyperopt.py](file://examples/hyperparam/search_hyperopt.py)
- [docker/MLproject](file://examples/docker/MLproject)
- [ray_serve/train_model.py](file://examples/ray_serve/train_model.py)
- [pyspark_ml_connect/pipeline.py](file://examples/pyspark_ml_connect/pipeline.py)
- [spark_udf/spark_udf.py](file://examples/spark_udf/spark_udf.py)
- [remote_store/remote_server.py](file://examples/remote_store/remote_server.py)
</cite>

## Table of Contents
1. [Multistep Workflows](#multistep-workflows)
2. [Hyperparameter Tuning with Hyperopt](#hyperparameter-tuning-with-hyperopt)
3. [Model Deployment with Docker](#model-deployment-with-docker)
4. [Model Deployment with Ray Serve](#model-deployment-with-ray-serve)
5. [Distributed Training with Spark](#distributed-training-with-spark)
6. [Remote Tracking Server Usage](#remote-tracking-server-usage)

## Multistep Workflows

The multistep workflow example demonstrates how to orchestrate complex machine learning pipelines using MLflow Projects. The workflow is defined in the MLproject file with multiple entry points that represent different stages of the pipeline: data loading, ETL, model training with ALS, and training a Keras neural network. The main.py script coordinates the execution of these steps, ensuring proper dependency management and artifact passing between stages. Each step runs as a separate MLflow run, with the main workflow run serving as the parent run, creating a hierarchical structure that makes it easy to track the entire pipeline execution.

**Section sources**
- [multistep_workflow/MLproject](file://examples/multistep_workflow/MLproject#L1-L38)
- [multistep_workflow/main.py](file://examples/multistep_workflow/main.py#L1-L108)

## Hyperparameter Tuning with Hyperopt

This use case demonstrates hyperparameter optimization using the Hyperopt library integrated with MLflow. The implementation creates a search space for hyperparameters such as learning rate and momentum, then uses Hyperopt's Tree of Parzen Estimators (TPE) algorithm to efficiently explore this space. Each hyperparameter configuration is evaluated in a separate MLflow run, with child runs nested under a parent run for organizational clarity. The optimization process automatically logs parameters, metrics, and artifacts for each trial, enabling comprehensive comparison of different hyperparameter configurations. The final run captures the best-performing configuration based on validation set performance.

**Section sources**
- [hyperparam/MLproject](file://examples/hyperparam/MLproject#L1-L60)
- [hyperparam/search_hyperopt.py](file://examples/hyperparam/search_hyperopt.py#L1-L170)

## Model Deployment with Docker

This example illustrates how to deploy MLflow models using Docker containers. The MLproject configuration specifies a Docker environment with a custom image name, allowing for consistent deployment across different environments. The entry point defines the command to execute the model training script with specified hyperparameters. This approach ensures that the model runs in an isolated environment with all dependencies properly configured, making it easy to deploy the same model across different infrastructure platforms. The Docker-based deployment provides reproducibility and eliminates "it works on my machine" issues.

**Section sources**
- [docker/MLproject](file://examples/docker/MLproject#L1-L12)

## Model Deployment with Ray Serve

This use case demonstrates integration between MLflow and Ray Serve for scalable model deployment. The example shows how to train a model using MLflow's autologging functionality, register the trained model in the MLflow Model Registry, and then deploy it using Ray Serve. The process begins with training a Gradient Boosting Classifier on the Iris dataset while automatically logging parameters, metrics, and the model artifact. The trained model is then registered in the Model Registry, creating a versioned record that can be deployed. This integration enables scalable, production-ready model serving with automatic scaling and load balancing capabilities provided by Ray Serve.

**Section sources**
- [ray_serve/train_model.py](file://examples/ray_serve/train_model.py#L1-L42)

## Distributed Training with Spark

This use case demonstrates distributed training and inference using Spark with MLflow integration. The examples show how to use MLflow to log Spark ML models, including pipelines with multiple stages such as feature scaling and classification. The pyspark_ml_connect example demonstrates how to train a logistic regression model using Spark Connect and log the resulting pipeline model to MLflow. The spark_udf example shows how to create a Spark UDF from a logged MLflow model, enabling scalable inference on large datasets. These examples illustrate how MLflow can be used to manage the lifecycle of distributed machine learning models, from training to deployment.

**Section sources**
- [pyspark_ml_connect/pipeline.py](file://examples/pyspark_ml_connect/pipeline.py#L1-L37)
- [spark_udf/spark_udf.py](file://examples/spark_udf/spark_udf.py#L1-L24)

## Remote Tracking Server Usage

This use case demonstrates how to use MLflow with a remote tracking server for centralized experiment management. The example shows how to configure MLflow to connect to a remote tracking server, enabling teams to share and compare experiments. The remote_server.py script illustrates the basic operations of logging parameters, metrics, and artifacts to a remote server. This setup allows multiple users and systems to record their experiments to a centralized location, providing a single source of truth for model development activities. The remote tracking server can be configured with various backend stores and artifact repositories, supporting scalable and secure collaboration across teams.

**Section sources**
- [remote_store/remote_server.py](file://examples/remote_store/remote_server.py#L1-L43)