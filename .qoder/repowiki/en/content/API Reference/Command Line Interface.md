# Command Line Interface

<cite>
**Referenced Files in This Document**   
- [mlflow/cli/__init__.py](file://mlflow/cli/__init__.py)
- [mlflow/models/cli.py](file://mlflow/models/cli.py)
- [mlflow/deployments/cli.py](file://mlflow/deployments/cli.py)
- [mlflow/cli/crypto.py](file://mlflow/cli/crypto.py)
- [mlflow/cli/eval.py](file://mlflow/cli/eval.py)
- [mlflow/cli/traces.py](file://mlflow/cli/traces.py)
- [mlflow/utils/cli_args.py](file://mlflow/utils/cli_args.py)
- [mlflow/environment_variables.py](file://mlflow/environment_variables.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Core Commands](#core-commands)
3. [Model Deployment Commands](#model-deployment-commands)
4. [Cryptographic Operations](#cryptographic-operations)
5. [Evaluation and Tracing Commands](#evaluation-and-tracing-commands)
6. [Configuration Options](#configuration-options)
7. [Common Workflows](#common-workflows)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Scripting and Automation](#scripting-and-automation)

## Introduction
The MLflow Command Line Interface (CLI) provides a comprehensive set of tools for managing machine learning experiments, models, and deployments. This documentation covers all available commands, subcommands, options, and arguments with their syntax and usage patterns. The CLI enables users to start the MLflow server, run projects, serve models, evaluate models, and perform specialized operations for evaluation, tracing, and cryptographic management.

The MLflow CLI is built on Click and follows standard command-line conventions with consistent option naming and help documentation. Commands are organized into a hierarchical structure with top-level commands and subcommands for specific functionality areas. Environment variables can be used to configure default values and authentication settings for CLI operations.

**Section sources**
- [mlflow/cli/__init__.py](file://mlflow/cli/__init__.py#L79-L90)

## Core Commands

### mlflow run
The `mlflow run` command executes MLflow projects from local directories or remote Git repositories. It supports various execution backends and configuration options.

```bash
mlflow run <URI> [OPTIONS]
```

**Arguments:**
- `URI`: Path to project (local directory or Git repository)

**Options:**
- `--entry-point`, `-e`: Entry point within project (default: main)
- `--version`, `-v`: Git commit reference for Git projects
- `--param-list`, `-P`: Parameters in NAME=VALUE format
- `--docker-args`, `-A`: Docker run arguments in NAME=VALUE format
- `--experiment-name`: Name of experiment for the run
- `--experiment-id`: ID of experiment for the run
- `--backend`: Execution backend (local, databricks, kubernetes)
- `--backend-config`: Path to JSON config file for backend
- `--env-manager`: Environment manager (local, virtualenv, conda, uv)
- `--storage-dir`: Directory for downloading artifacts (local backend only)
- `--run-id`: Use existing run ID instead of creating new run
- `--run-name`: Name for the MLflow Run
- `--build-image`: Build Docker image for Docker projects

**Examples:**
```bash
# Run project from local directory
mlflow run . --entry-point train --param-list learning_rate=0.01

# Run project from Git repository
mlflow run https://github.com/mlflow/mlflow-example --version v1.0

# Run with Databricks backend
mlflow run . --backend databricks --backend-config cluster.json
```

**Section sources**
- [mlflow/cli/__init__.py](file://mlflow/cli/__init__.py#L94-L214)

### mlflow server
The `mlflow server` command starts the MLflow tracking server with built-in security middleware. The server provides a web interface and REST API for tracking experiments and managing models.

```bash
mlflow server [OPTIONS]
```

**Options:**
- `--backend-store-uri`: URI for experiment and run data persistence
- `--registry-store-uri`: URI for registered models persistence
- `--default-artifact-root`: Directory for storing artifacts for new experiments
- `--serve-artifacts`: Enable artifact serving through proxy
- `--artifacts-only`: Configure server for artifact serving only
- `--artifacts-destination`: Base artifact location for upload/download requests
- `--host`: Network interface to bind server to (default: 127.0.0.1)
- `--port`: Port to listen on (default: 5000)
- `--workers`: Number of worker processes
- `--allowed-hosts`: Comma-separated list of allowed Host headers
- `--cors-allowed-origins`: Comma-separated list of allowed CORS origins
- `--disable-security-middleware`: Disable all security middleware
- `--x-frame-options`: X-Frame-Options header value (SAMEORIGIN, DENY, NONE)
- `--static-prefix`: Prefix for all static paths
- `--gunicorn-opts`: Additional options for gunicorn processes
- `--waitress-opts`: Additional options for waitress-serve
- `--uvicorn-opts`: Additional options for uvicorn processes
- `--expose-prometheus`: Directory path for Prometheus metrics
- `--app-name`: Application name for tracking server
- `--dev`: Run server with debug logging and auto-reload
- `--secrets-cache-ttl`: Server-side secrets cache time-to-live in seconds
- `--secrets-cache-max-size`: Server-side secrets cache maximum entries

**Examples:**
```bash
# Start server with default settings
mlflow server --host 0.0.0.0 --port 5000

# Start server with database backend
mlflow server --backend-store-uri sqlite:///mlflow.db --default-artifact-root ./artifacts

# Start server with security configuration
mlflow server --host 0.0.0.0 --allowed-hosts mlflow.company.com --cors-allowed-origins https://app.company.com
```

**Section sources**
- [mlflow/cli/__init__.py](file://mlflow/cli/__init__.py#L354-L509)

### mlflow gc
The `mlflow gc` command permanently deletes runs in the "deleted" lifecycle stage from the backend store. This operation removes all associated metadata, artifacts, and model data.

```bash
mlflow gc [OPTIONS]
```

**Options:**
- `--older-than`: Remove runs older than specified time limit (format: #d#h#m#s)
- `--backend-store-uri`: URI of backend store from which to delete runs
- `--artifacts-destination`: Base artifact location for resolving artifact requests
- `--run-ids`: Comma-separated list of runs to delete
- `--experiment-ids`: Comma-separated list of experiments to delete
- `--logged-model-ids`: Comma-separated list of logged model IDs to delete
- `--tracking-uri`: Tracking URI for deleting deleted runs

**Examples:**
```bash
# Delete runs older than 30 days
mlflow gc --older-than 30d

# Delete specific runs by ID
mlflow gc --run-ids 'run1,run2,run3'

# Delete all runs in specific experiments
mlflow gc --experiment-ids 'exp1,exp2'
```

**Section sources**
- [mlflow/cli/__init__.py](file://mlflow/cli/__init__.py#L651-L715)

## Model Deployment Commands

### mlflow models
The `mlflow models` command group provides tools for deploying MLflow models locally and managing model environments.

#### mlflow models serve
Serves a saved MLflow model by launching a web server on the specified host and port.

```bash
mlflow models serve [OPTIONS]
```

**Options:**
- `--model-uri`, `-m`: URI to the model
- `--port`, `-p`: Port to listen on (default: 5000)
- `--host`, `-h`: Host to bind to (default: 127.0.0.1)
- `--timeout`: Timeout in seconds for serving requests (default: 60)
- `--workers`, `-w`: Number of workers to handle requests
- `--env-manager`: Environment manager (local, virtualenv, conda, uv)
- `--no-conda`: Use local environment instead of conda
- `--install-mlflow`: Install MLflow in the environment
- `--enable-mlserver`: Enable serving with MLServer through v2 inference protocol

**Examples:**
```bash
# Serve model from run
mlflow models serve -m runs:/my-run-id/model-path --port 1234

# Serve model with custom configuration
mlflow models serve -m models:/my-model/production --port 8080 --workers 4
```

**Section sources**
- [mlflow/models/cli.py](file://mlflow/models/cli.py#L24-L106)

#### mlflow models predict
Generates predictions in JSON format using a saved MLflow model.

```bash
mlflow models predict [OPTIONS]
```

**Options:**
- `--model-uri`, `-m`: URI to the model
- `--input-path`, `-i`: CSV file containing pandas DataFrame for prediction
- `--output-path`, `-o`: File to output results (default: stdout)
- `--content-type`, `-t`: Content type of input file (json, csv)
- `--env-manager`: Environment manager
- `--install-mlflow`: Install MLflow in the environment
- `--pip-requirements-override`, `-r`: Packages to override dependencies
- `--env`: Extra environment variables (key=value)

**Examples:**
```bash
# Predict with JSON input
mlflow models predict -m runs:/my-run-id/model-path -i input.json

# Predict with CSV input and output to file
mlflow models predict -m models:/my-model/latest -i data.csv -o predictions.json
```

**Section sources**
- [mlflow/models/cli.py](file://mlflow/models/cli.py#L118-L178)

#### mlflow models build-docker
Builds a Docker image whose default entrypoint serves an MLflow model at port 8080.

```bash
mlflow models build-docker [OPTIONS]
```

**Options:**
- `--model-uri`, `-m`: URI to the model
- `--name`, `-n`: Name for built image (default: mlflow-pyfunc-servable)
- `--env-manager`: Environment manager
- `--mlflow-home`: Path to local MLflow project clone
- `--install-java`: Install Java in the image
- `--install-mlflow`: Install MLflow in the environment
- `--enable-mlserver`: Enable MLServer in the Docker image

**Examples:**
```bash
# Build Docker image with model
mlflow models build-docker --model-uri "runs:/some-run-uuid/my-model" --name "my-image-name"

# Build generic Docker image
mlflow models build-docker --name "generic-model-server"
```

**Section sources**
- [mlflow/models/cli.py](file://mlflow/models/cli.py#L251-L311)

## Model Deployment Commands

### mlflow deployments
The `mlflow deployments` command group manages model deployments to various target platforms.

#### mlflow deployments create
Deploys a model to a specified target platform.

```bash
mlflow deployments create [OPTIONS]
```

**Options:**
- `--target`, `-t`: Deployment target URI
- `--name`: Name of the deployment
- `--model-uri`, `-m`: URI to the model
- `--flavor`, `-f`: Model flavor to deploy
- `--config`, `-C`: Target-specific config in NAME=VALUE format

**Examples:**
```bash
# Create deployment to local target
mlflow deployments create -t local -m runs:/my-run-id/model-path --name my-model

# Create deployment with configuration
mlflow deployments create -t sagemaker -m models:/my-model/production --name prod-model -C instance_type=m5.large
```

**Section sources**
- [mlflow/deployments/cli.py](file://mlflow/deployments/cli.py#L123-L150)

#### mlflow deployments update
Updates an existing model deployment with a new model or configuration.

```bash
mlflow deployments update [OPTIONS]
```

**Options:**
- `--target`, `-t`: Deployment target URI
- `--name`: Name of the deployment
- `--model-uri`, `-m`: New model URI
- `--flavor`, `-f`: Model flavor to deploy
- `--config`, `-C`: Target-specific config in NAME=VALUE format

**Examples:**
```bash
# Update deployment with new model
mlflow deployments update -t local --name my-model --model-uri runs:/new-run-id/model-path
```

**Section sources**
- [mlflow/deployments/cli.py](file://mlflow/deployments/cli.py#L153-L192)

#### mlflow deployments delete
Deletes a model deployment from the specified target.

```bash
mlflow deployments delete [OPTIONS]
```

**Options:**
- `--target`, `-t`: Deployment target URI
- `--name`: Name of the deployment
- `--config`, `-C`: Target-specific config in NAME=VALUE format

**Examples:**
```bash
# Delete deployment
mlflow deployments delete -t local --name my-model
```

**Section sources**
- [mlflow/deployments/cli.py](file://mlflow/deployments/cli.py#L195-L219)

#### mlflow deployments predict
Generates predictions using a deployed model.

```bash
mlflow deployments predict [OPTIONS]
```

**Options:**
- `--target`, `-t`: Deployment target URI
- `--name`: Name of the deployment
- `--endpoint`: Name of the endpoint
- `--input-path`, `-I`: Path to input prediction payload file
- `--output-path`, `-O`: File to output results

**Examples:**
```bash
# Predict with deployed model
mlflow deployments predict -t local --name my-model -I input.json -O output.json
```

**Section sources**
- [mlflow/deployments/cli.py](file://mlflow/deployments/cli.py#L294-L327)

## Cryptographic Operations

### mlflow crypto
The `mlflow crypto` command group manages cryptographic operations for MLflow's secure storage of sensitive information.

#### mlflow crypto rotate-kek
Rotates the KEK (Key Encryption Key) passphrase used for encrypting and decrypting sensitive data in the MLflow backend store.

```bash
mlflow crypto rotate-kek [OPTIONS]
```

**Options:**
- `--new-passphrase`: New KEK passphrase (required, prompted securely)
- `--backend-store-uri`: URI of the backend store
- `--yes`, `-y`: Skip confirmation prompt

**Workflow:**
1. Shut down the MLflow server
2. Set MLFLOW_CRYPTO_KEK_PASSPHRASE to the current (old) passphrase
3. Set MLFLOW_CRYPTO_KEK_VERSION to the current version
4. Run the rotate-kek command with the new passphrase
5. Update deployment configuration with new passphrase and incremented version
6. Restart the MLflow server

**Examples:**
```bash
# Rotate KEK passphrase
mlflow crypto rotate-kek --new-passphrase
```

**Section sources**
- [mlflow/cli/crypto.py](file://mlflow/cli/crypto.py#L24-L208)

## Evaluation and Tracing Commands

### mlflow traces
The `mlflow traces` command group manages trace data, assessments, and metadata for MLflow tracing.

#### mlflow traces search
Searches for traces with filtering, sorting, and field selection capabilities.

```bash
mlflow traces search [OPTIONS]
```

**Options:**
- `--experiment-id`, `-x`: Experiment ID to search within
- `--filter-string`: Filter string for trace search
- `--max-results`: Maximum number of traces to return (default: 100)
- `--order-by`: Fields to order by
- `--page-token`: Token for pagination
- `--run-id`: Filter by run ID
- `--include-spans/--no-include-spans`: Include span data in results
- `--model-id`: Filter by model ID
- `--output`: Output format (table, json)
- `--extract-fields`: Select specific fields using dot notation
- `--verbose`: Show all available fields in error messages

**Examples:**
```bash
# Search traces in experiment
mlflow traces search --experiment-id 1 --max-results 50

# Filter traces by status and timestamp
mlflow traces search --experiment-id 1 --filter-string "status = 'OK' AND timestamp_ms > 1700000000000"

# Get specific fields in JSON format
mlflow traces search --experiment-id 1 --extract-fields "info.trace_id,info.assessments.*" --output json
```

**Section sources**
- [mlflow/cli/traces.py](file://mlflow/cli/traces.py#L167-L371)

#### mlflow traces log-feedback
Logs evaluation feedback or scores to traces.

```bash
mlflow traces log-feedback [OPTIONS]
```

**Options:**
- `--trace-id`: Trace ID to log feedback to
- `--name`: Feedback name
- `--value`: Feedback value
- `--source-type`: Source type (HUMAN, LLM_JUDGE, CODE)
- `--source-id`: Source identifier
- `--rationale`: Explanation for the feedback
- `--metadata`: Additional metadata as JSON string
- `--span-id`: Associate feedback with a specific span ID

**Examples:**
```bash
# Log numeric feedback
mlflow traces log-feedback --trace-id tr-abc123 --name relevance --value 0.9 --rationale "Highly relevant response"

# Log human feedback
mlflow traces log-feedback --trace-id tr-abc123 --name quality --value good --source-type HUMAN --source-id reviewer@example.com
```

**Section sources**
- [mlflow/cli/traces.py](file://mlflow/cli/traces.py#L505-L604)

### mlflow evaluate
The `mlflow evaluate` command evaluates traces with specified scorers and outputs results.

```bash
mlflow evaluate [OPTIONS]
```

**Options:**
- `--experiment-id`: Experiment ID for evaluation
- `--trace-ids`: Comma-separated list of trace IDs to evaluate
- `--scorers`: Comma-separated list of scorer names
- `--output-format`: Output format (table, json)

**Examples:**
```bash
# Evaluate single trace
mlflow evaluate --experiment-id 1 --trace-ids tr-abc123 --scorers Correctness,Safety

# Evaluate multiple traces with JSON output
mlflow evaluate --experiment-id 1 --trace-ids tr-abc123,tr-abc124 --scorers Relevance --output-format json
```

**Section sources**
- [mlflow/cli/eval.py](file://mlflow/cli/eval.py#L61-L129)

## Configuration Options

### Environment Variables
MLflow CLI commands can be configured using environment variables that control various aspects of behavior, authentication, and performance.

**Tracking Configuration:**
- `MLFLOW_TRACKING_URI`: Specifies the tracking URI
- `MLFLOW_TRACKING_USERNAME`: Username for tracking server authentication
- `MLFLOW_TRACKING_PASSWORD`: Password for tracking server authentication
- `MLFLOW_TRACKING_TOKEN`: Token for tracking server authentication
- `MLFLOW_TRACKING_INSECURE_TLS`: Verify TLS connection
- `MLFLOW_TRACKING_SERVER_CERT_PATH`: Path to server certificate
- `MLFLOW_TRACKING_CLIENT_CERT_PATH`: Path to client certificate

**Experiment Configuration:**
- `MLFLOW_EXPERIMENT_ID`: Default experiment ID
- `MLFLOW_EXPERIMENT_NAME`: Default experiment name
- `MLFLOW_RUN_ID`: ID of the run to log data to

**Artifact Storage Configuration:**
- `MLFLOW_S3_ENDPOINT_URL`: S3 endpoint URL for artifact operations
- `MLFLOW_S3_IGNORE_TLS`: Skip TLS certificate verification for S3
- `MLFLOW_GCS_DOWNLOAD_CHUNK_SIZE`: Chunk size for GCS downloads
- `MLFLOW_GCS_UPLOAD_CHUNK_SIZE`: Chunk size for GCS uploads

**Performance and Timeout Configuration:**
- `MLFLOW_HTTP_REQUEST_TIMEOUT`: Timeout for HTTP requests (default: 120)
- `MLFLOW_SCORING_SERVER_REQUEST_TIMEOUT`: Scoring server request timeout (default: 60)
- `MLFLOW_REQUIREMENTS_INFERENCE_TIMEOUT`: Model dependency inference timeout (default: 120)
- `MLFLOW_INPUT_EXAMPLE_INFERENCE_TIMEOUT`: Input example inference timeout (default: 180)

**Security Configuration:**
- `MLFLOW_SERVER_ALLOWED_HOSTS`: Allowed Host headers to prevent DNS rebinding
- `MLFLOW_SERVER_CORS_ALLOWED_ORIGINS`: Allowed CORS origins
- `MLFLOW_SERVER_X_FRAME_OPTIONS`: X-Frame-Options header value
- `MLFLOW_CRYPTO_KEK_PASSPHRASE`: KEK passphrase for encrypted data
- `MLFLOW_CRYPTO_KEK_VERSION`: KEK version for encrypted data

**Deployment Configuration:**
- `MLFLOW_DEPLOYMENT_PREDICT_TIMEOUT`: Timeout for deployment predict requests
- `MLFLOW_DEPLOYMENT_PREDICT_TOTAL_TIMEOUT`: Total timeout for deployment retries
- `MLFLOW_ENABLE_MULTIPART_UPLOAD`: Enable multipart upload for large artifacts
- `MLFLOW_ENABLE_MULTIPART_DOWNLOAD`: Enable multipart download for large files

**Section sources**
- [mlflow/environment_variables.py](file://mlflow/environment_variables.py#L101-L800)

## Common Workflows

### Starting the MLflow Server
The MLflow server provides a web interface and REST API for tracking experiments and managing models. Here are common server startup patterns:

**Basic Server:**
```bash
mlflow server --host 0.0.0.0 --port 5000
```

**Server with Database Backend:**
```bash
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./artifacts \
  --host 0.0.0.0 \
  --port 5000
```

**Secure Server Configuration:**
```bash
mlflow server \
  --host 0.0.0.0 \
  --allowed-hosts mlflow.company.com,app.company.com \
  --cors-allowed-origins https://app.company.com,https://notebook.company.com \
  --x-frame-options SAMEORIGIN \
  --port 5000
```

### Running Projects
MLflow projects provide a format for packaging data science code in a reusable and reproducible way.

**Local Project Execution:**
```bash
mlflow run . --entry-point train --param-list learning_rate=0.01 batch_size=32
```

**Git Repository Project:**
```bash
mlflow run https://github.com/mlflow/mlflow-example --version v1.0 --experiment-name "My Experiment"
```

**Docker-based Project:**
```bash
mlflow run . --backend local --env-manager docker --build-image
```

### Serving Models
Model serving allows you to deploy trained models as REST APIs for inference.

**Local Model Serving:**
```bash
mlflow models serve \
  --model-uri runs:/my-run-id/model-path \
  --port 1234 \
  --host 0.0.0.0 \
  --workers 4
```

**Docker-based Model Serving:**
```bash
# Build Docker image
mlflow models build-docker \
  --model-uri runs:/my-run-id/model-path \
  --name my-model-server

# Run container
docker run -p 8080:8080 my-model-server
```

### Evaluating Models
Model evaluation provides tools for assessing model performance and quality.

**Trace Evaluation:**
```bash
mlflow evaluate \
  --experiment-id 1 \
  --trace-ids tr-abc123,tr-def456 \
  --scorers Correctness,Safety,Relevance \
  --output-format table
```

**Batch Prediction Evaluation:**
```bash
mlflow models predict \
  --model-uri models:/my-model/production \
  --input-path test_data.json \
  --output-path predictions.json
```

## Troubleshooting Guide

### Command Not Found
If the `mlflow` command is not found, ensure MLflow is properly installed:

```bash
# Install MLflow
pip install mlflow

# Verify installation
mlflow --version
```

### Permission Errors
Permission errors typically occur when the MLflow server cannot write to the specified directories:

**Solutions:**
- Ensure the user has write permissions to the backend store directory
- Check file ownership and permissions
- Use appropriate directories with proper access rights

```bash
# Check directory permissions
ls -la ./mlruns

# Change ownership if needed
sudo chown -R $USER:$USER ./mlruns
```

### Network Connectivity Problems
Network issues can prevent the MLflow server from being accessible:

**Common Issues and Solutions:**
- **Connection refused**: Ensure the server is running and listening on the correct port
- **Timeout errors**: Check firewall settings and network connectivity
- **DNS resolution**: Verify hostnames are resolvable

```bash
# Check if server is listening
netstat -tlnp | grep 5000

# Test connectivity
curl http://localhost:5000

# Check firewall rules
sudo ufw status
```

### Authentication Issues
Authentication problems occur when connecting to secured MLflow servers:

**Solutions:**
- Verify tracking URI includes authentication credentials
- Check environment variables for correct values
- Ensure tokens are not expired

```bash
# Set tracking URI with authentication
export MLFLOW_TRACKING_URI=https://username:password@mlflow.company.com

# Or use token authentication
export MLFLOW_TRACKING_TOKEN=your-api-token
```

### Artifact Storage Issues
Problems with artifact storage often relate to configuration or permissions:

**Common Issues:**
- **S3 access denied**: Verify IAM permissions and credentials
- **GCS authentication**: Ensure service account has proper roles
- **Local storage full**: Check disk space and cleanup old artifacts

```bash
# Check disk space
df -h

# Set S3 endpoint if using non-AWS S3-compatible storage
export MLFLOW_S3_ENDPOINT_URL=https://s3.example.com
```

## Scripting and Automation

### CI/CD Pipeline Integration
MLflow CLI commands can be integrated into CI/CD pipelines for automated model training and deployment.

**GitHub Actions Example:**
```yaml
name: MLflow Pipeline
on: [push]
jobs:
  train-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: 3.8
    - name: Install MLflow
      run: pip install mlflow
    - name: Train model
      run: mlflow run . --experiment-name "CI/CD Training"
    - name: Deploy model
      if: github.ref == 'refs/heads/main'
      run: |
        mlflow deployments create \
          -t local \
          -m "models:/MyModel/Production" \
          --name my-production-model
```

**Jenkins Pipeline Example:**
```groovy
pipeline {
    agent any
    stages {
        stage('Train Model') {
            steps {
                sh 'mlflow run . --experiment-name "Jenkins Training"'
            }
        }
        stage('Evaluate Model') {
            steps {
                script {
                    def runId = sh(script: 'mlflow run . --experiment-name "Evaluation" | grep "Run ID" | cut -d"'"'"'" -f2', returnStdout: true).trim()
                    sh "mlflow evaluate --experiment-id 1 --trace-ids ${runId} --scorers Accuracy"
                }
            }
        }
        stage('Deploy Model') {
            steps {
                sh '''
                mlflow deployments create \
                  -t sagemaker \
                  -m "models:/MyModel/Production" \
                  --name production-endpoint \
                  -C instance_type=m5.large
                '''
            }
        }
    }
}
```

### Automation Scripts
Automation scripts can combine multiple MLflow commands for complex workflows.

**Model Training and Registration Script:**
```bash
#!/bin/bash
set -e

# Train model and get run ID
RUN_ID=$(mlflow run . --entry-point train --param-list learning_rate=0.01 | grep "Run ID" | cut -d"'"'"'" -f2)

# Wait for run to complete
sleep 30

# Register model
mlflow models serve -m "runs:/${RUN_ID}/model" --name my-model --stage Production

echo "Model registered with run ID: ${RUN_ID}"
```

**Batch Prediction Script:**
```bash
#!/bin/bash
set -e

# Process multiple input files
for input_file in data/batch_*.json; do
    output_file="results/$(basename ${input_file})"
    mlflow models predict \
      --model-uri models:/my-model/production \
      --input-path ${input_file} \
      --output-path ${output_file}
    
    echo "Processed ${input_file} -> ${output_file}"
done
```

**Scheduled Evaluation Script:**
```bash
#!/bin/bash
set -e

# Get recent runs for evaluation
RUN_IDS=$(mlflow runs list --experiment-id 1 --view active --max-results 5 | grep "RUN_ID" | awk '{print $2}' | tr '\n' ',' | sed 's/,$//')

# Evaluate runs
if [ -n "$RUN_IDS" ]; then
    mlflow evaluate \
      --experiment-id 1 \
      --trace-ids ${RUN_IDS} \
      --scorers Accuracy,Precision,Recall \
      --output-format json > evaluation_results.json
    
    echo "Evaluation completed for runs: ${RUN_IDS}"
else
    echo "No runs found for evaluation"
fi
```

**Section sources**
- [mlflow/cli/__init__.py](file://mlflow/cli/__init__.py#L79-L90)
- [mlflow/models/cli.py](file://mlflow/models/cli.py#L24-L106)
- [mlflow/deployments/cli.py](file://mlflow/deployments/cli.py#L123-L150)
- [mlflow/cli/crypto.py](file://mlflow/cli/crypto.py#L24-L208)
- [mlflow/cli/eval.py](file://mlflow/cli/eval.py#L61-L129)
- [mlflow/cli/traces.py](file://mlflow/cli/traces.py#L167-L371)
- [mlflow/utils/cli_args.py](file://mlflow/utils/cli_args.py#L12-L313)
- [mlflow/environment_variables.py](file://mlflow/environment_variables.py#L101-L800)