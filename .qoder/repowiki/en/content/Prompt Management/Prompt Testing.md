# Prompt Testing

<cite>
**Referenced Files in This Document**   
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [optimize.py](file://mlflow/genai/optimize/optimize.py)
- [make_judge.py](file://mlflow/genai/judges/make_judge.py)
- [base.py](file://mlflow/genai/scorers/base.py)
- [evaluation_dataset.py](file://mlflow/genai/datasets/evaluation_dataset.py)
- [entities.py](file://mlflow/genai/evaluation/entities.py)
- [utils.py](file://mlflow/genai/evaluation/utils.py)
- [evaluate_with_llm_judge.py](file://examples/evaluation/evaluate_with_llm_judge.py)
- [evaluate_with_static_dataset.py](file://examples/evaluation/evaluate_with_static_dataset.py)
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
Prompt testing in MLflow provides a systematic framework for evaluating and optimizing prompts through automated testing. This system enables developers to rigorously assess prompt quality, compare different prompt versions, and leverage LLM judges for objective evaluation. The implementation supports defining test datasets, evaluating prompts against these datasets, measuring results, and comparing performance across iterations. The interfaces allow creating evaluation harnesses, defining test metrics, and running comparative tests between prompt variants. This documentation explains the implementation details, relationships with other components like evaluation metrics and tracing systems, and addresses common issues in prompt testing.

## Project Structure
The prompt testing functionality in MLflow is organized within the `genai` module, which contains specialized components for evaluation, optimization, and dataset management. The core components are located in subdirectories that handle specific aspects of prompt testing:

- `genai/evaluation/`: Contains the evaluation harness and related utilities
- `genai/optimize/`: Houses prompt optimization algorithms and utilities
- `genai/judges/`: Implements LLM judge functionality for evaluation
- `genai/scorers/`: Defines scoring mechanisms and metrics
- `genai/datasets/`: Manages evaluation datasets
- `examples/evaluation/`: Provides practical examples of evaluation usage

This structure supports a comprehensive workflow from dataset creation to prompt optimization and evaluation.

```mermaid
graph TD
subgraph "Core Evaluation Components"
harness[harness.py]
entities[entities.py]
utils[utils.py]
end
subgraph "Optimization Components"
optimize[optimize.py]
optimizers[optimizers/]
end
subgraph "Judges and Scorers"
judges[judges/]
scorers[scorers/]
end
subgraph "Dataset Management"
datasets[datasets/]
evaluation_dataset[evaluation_dataset.py]
end
subgraph "Examples"
examples[examples/evaluation/]
llm_judge[evaluate_with_llm_judge.py]
static_dataset[evaluate_with_static_dataset.py]
end
harness --> entities
harness --> utils
optimize --> harness
judges --> scorers
datasets --> harness
examples --> harness
examples --> judges
```

**Diagram sources**
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [optimize.py](file://mlflow/genai/optimize/optimize.py)
- [make_judge.py](file://mlflow/genai/judges/make_judge.py)
- [base.py](file://mlflow/genai/scorers/base.py)
- [evaluation_dataset.py](file://mlflow/genai/datasets/evaluation_dataset.py)

**Section sources**
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [optimize.py](file://mlflow/genai/optimize/optimize.py)
- [make_judge.py](file://mlflow/genai/judges/make_judge.py)
- [base.py](file://mlflow/genai/scorers/base.py)
- [evaluation_dataset.py](file://mlflow/genai/datasets/evaluation_dataset.py)

## Core Components
The prompt testing system in MLflow consists of several core components that work together to enable systematic evaluation and optimization of prompts. The evaluation harness serves as the central component, coordinating the evaluation process by managing datasets, executing predictions, and computing scores. The optimization framework builds upon this foundation to automatically improve prompt quality based on evaluation metrics. LLM judges provide sophisticated evaluation capabilities by using language models to assess prompt quality, while the scorer system offers flexible metric definition and computation.

The system supports both static datasets and dynamic evaluation using traces, allowing for comprehensive testing across different scenarios. The architecture is designed to be extensible, enabling custom scorers and evaluation metrics while maintaining compatibility with MLflow's tracking and logging capabilities.

**Section sources**
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [optimize.py](file://mlflow/genai/optimize/optimize.py)
- [make_judge.py](file://mlflow/genai/judges/make_judge.py)
- [base.py](file://mlflow/genai/scorers/base.py)

## Architecture Overview
The prompt testing architecture in MLflow follows a modular design that separates concerns while enabling seamless integration between components. The system operates on a pipeline model where datasets flow through evaluation stages, with results being aggregated and analyzed at each step.

```mermaid
graph LR
Dataset[Evaluation Dataset] --> Harness[Evaluation Harness]
Harness --> Predict[Prediction Function]
Harness --> Scorers[Scorers]
Scorers --> Metrics[Metrics]
Harness --> Tracing[Tracing System]
Tracing --> Results[Evaluation Results]
Results --> Optimization[Prompt Optimization]
Optimization --> ImprovedPrompts[Improved Prompts]
ImprovedPrompts --> Harness
style Dataset fill:#f9f,stroke:#333
style Harness fill:#bbf,stroke:#333
style Predict fill:#f96,stroke:#333
style Scorers fill:#9f9,stroke:#333
style Metrics fill:#ff9,stroke:#333
style Tracing fill:#9ff,stroke:#333
style Results fill:#f9f,stroke:#333
style Optimization fill:#f96,stroke:#333
style ImprovedPrompts fill:#9f9,stroke:#333
```

**Diagram sources**
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [optimize.py](file://mlflow/genai/optimize/optimize.py)
- [utils.py](file://mlflow/genai/evaluation/utils.py)

## Detailed Component Analysis

### Evaluation Harness
The evaluation harness is the core component that orchestrates the prompt testing process. It manages the evaluation workflow by coordinating dataset processing, prediction execution, and score computation. The harness operates in parallel across multiple evaluation items, using thread pools to maximize efficiency while maintaining isolation between evaluation tasks.

The harness supports both single-turn and multi-turn evaluations, with special handling for session-based assessments. It integrates with MLflow's tracing system to capture execution details and provides comprehensive progress tracking during evaluation. The component is designed to handle various input formats, including pandas DataFrames, Spark DataFrames, and lists of dictionaries, making it flexible for different use cases.

#### For Object-Oriented Components:
```mermaid
classDiagram
class EvalItem {
+request_id : str
+inputs : dict[str, Any]
+outputs : Any
+expectations : dict[str, Any]
+tags : dict[str, str]
+trace : Trace
+error_message : str
+source : DatasetRecordSource
+from_trace(trace : Trace) EvalItem
+from_dataset_row(row : dict[str, Any]) EvalItem
+get_expectation_assessments() list[Expectation]
+to_dict() dict[str, Any]
}
class EvalResult {
+eval_item : EvalItem
+assessments : list[Feedback]
+eval_error : str
+to_pd_series() pd.Series
+get_assessments_dict() dict[str, Any]
}
class EvaluationResult {
+run_id : str
+metrics : dict[str, float]
+result_df : pd.DataFrame
+__repr__() str
+tables : dict[str, pd.DataFrame]
}
class EvaluationHarness {
+run(eval_df : pd.DataFrame, predict_fn : Callable, scorers : list[Scorer], run_id : str) EvaluationResult
+_run_single(eval_item : EvalItem, scorers : list[Scorer], run_id : str, predict_fn : Callable) EvalResult
+_compute_eval_scores(eval_item : EvalItem, scorers : list[Scorer]) list[Feedback]
+_log_assessments(run_id : str, trace : Trace, assessments : list[Assessment]) void
+_refresh_eval_result_traces(eval_results : list[EvalResult]) void
}
EvaluationHarness --> EvalItem : "processes"
EvaluationHarness --> EvalResult : "produces"
EvaluationHarness --> EvaluationResult : "returns"
EvalResult --> EvalItem : "contains"
EvalResult --> Feedback : "contains"
```

**Diagram sources**
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [entities.py](file://mlflow/genai/evaluation/entities.py)

### Prompt Optimization Framework
The prompt optimization framework provides automated methods for improving prompt quality based on evaluation metrics. It uses optimization algorithms to iteratively refine prompts, leveraging evaluation results to guide the improvement process. The framework supports various optimization strategies and allows for custom objective functions that combine multiple evaluation metrics.

The optimization process is designed to work seamlessly with MLflow's tracking system, automatically logging optimization progress, intermediate results, and final outcomes. This enables reproducible optimization runs and facilitates comparison between different optimization approaches.

#### For Object-Oriented Components:
```mermaid
classDiagram
class PromptOptimizerOutput {
+optimized_prompts : dict[str, str]
+initial_eval_score : float
+final_eval_score : float
}
class PromptOptimizationResult {
+optimized_prompts : list[PromptVersion]
+optimizer_name : str
+initial_eval_score : float
+final_eval_score : float
}
class BasePromptOptimizer {
+model_name : str
+optimize(eval_fn : Callable, dataset : list[dict], target_prompts : dict, enable_tracking : bool) PromptOptimizerOutput
}
class GepaPromptOptimizer {
+reflection_model : str
+optimize(eval_fn : Callable, dataset : list[dict], target_prompts : dict, enable_tracking : bool) PromptOptimizerOutput
}
class PromptOptimizationSystem {
+optimize_prompts(predict_fn : Callable, train_data : EvaluationDatasetTypes, prompt_uris : list[str], optimizer : BasePromptOptimizer, scorers : list[Scorer], aggregation : AggregationFn, enable_tracking : bool) PromptOptimizationResult
+_build_eval_fn(predict_fn : Callable, metric_fn : Callable) Callable
}
PromptOptimizationSystem --> BasePromptOptimizer : "uses"
PromptOptimizationSystem --> GepaPromptOptimizer : "can use"
PromptOptimizationSystem --> PromptOptimizationResult : "returns"
BasePromptOptimizer --> PromptOptimizerOutput : "returns"
GepaPromptOptimizer --> PromptOptimizerOutput : "returns"
```

**Diagram sources**
- [optimize.py](file://mlflow/genai/optimize/optimize.py)

### LLM Judges and Scoring System
The LLM judges and scoring system provides sophisticated evaluation capabilities by leveraging language models to assess prompt quality. The system supports both built-in metrics and custom scorers, allowing for flexible evaluation strategies. LLM judges can be configured with specific instructions and models to perform targeted evaluations, such as assessing response quality, correctness, or adherence to guidelines.

The scoring system is designed to be extensible, enabling developers to create custom scorers using decorators or by implementing the Scorer interface. The system handles various data types and provides aggregation functions to compute overall performance metrics from individual scores.

#### For API/Service Components:
```mermaid
sequenceDiagram
participant User as "User Application"
participant Harness as "Evaluation Harness"
participant Judge as "LLM Judge"
participant Model as "Target LLM"
participant Scorer as "Custom Scorer"
User->>Harness : Provide dataset and scorers
Harness->>Model : Execute predictions
Harness->>Judge : Send inputs, outputs, expectations
Judge->>Judge : Process evaluation request
Judge->>Model : Query for assessment
Model-->>Judge : Return assessment
Judge-->>Harness : Return score and rationale
Harness->>Scorer : Execute custom scorer
Scorer-->>Harness : Return score
Harness->>Harness : Aggregate scores
Harness-->>User : Return evaluation results
```

**Diagram sources**
- [make_judge.py](file://mlflow/genai/judges/make_judge.py)
- [base.py](file://mlflow/genai/scorers/base.py)

### Dataset Management System
The dataset management system provides a unified interface for handling evaluation datasets in MLflow. It supports both standard MLflow evaluation datasets and Databricks managed datasets, offering a consistent API across different storage backends. The system handles dataset creation, merging, and conversion to various formats, making it easy to work with evaluation data.

The dataset system integrates with MLflow's tracking capabilities, allowing datasets to be logged and versioned alongside models and experiments. It supports various data formats and provides utilities for data validation and preprocessing, ensuring that evaluation datasets are properly formatted for testing.

#### For Object-Oriented Components:
```mermaid
classDiagram
class EvaluationDataset {
+_databricks_dataset : Dataset
+_mlflow_dataset : EvaluationDataset
+_df : DataFrame
+digest : str
+name : str
+dataset_id : str
+source : DatasetSource
+source_type : str
+created_time : int
+tags : dict[str, Any]
+experiment_ids : list[str]
+schema : str
+profile : str
+set_profile(profile : str) EvaluationDataset
+merge_records(records : list[dict]) EvaluationDataset
+to_df() DataFrame
+has_records() bool
+to_dict() dict[str, Any]
+from_dict(data : dict) EvaluationDataset
+to_proto() Proto
+from_proto(proto) EvaluationDataset
+to_evaluation_dataset(path : str, feature_names : list[str]) LegacyEvaluationDataset
+_to_mlflow_entity() DatasetEntity
}
class DatasetSource {
+table_name : str
+dataset_id : str
}
class LegacyEvaluationDataset {
+data : DataFrame
+path : str
+feature_names : list[str]
+name : str
+digest : str
}
class DatasetEntity {
+name : str
+digest : str
+source_type : str
+source : str
+schema : str
+profile : str
}
EvaluationDataset --> DatasetSource : "has"
EvaluationDataset --> LegacyEvaluationDataset : "converts to"
EvaluationDataset --> DatasetEntity : "converts to"
```

**Diagram sources**
- [evaluation_dataset.py](file://mlflow/genai/datasets/evaluation_dataset.py)

## Dependency Analysis
The prompt testing system in MLflow has a well-defined dependency structure that enables modularity while maintaining integration between components. The evaluation harness serves as the central component, depending on various subsystems for specific functionality.

```mermaid
graph TD
harness[harness.py] --> entities[entities.py]
harness --> utils[utils.py]
harness --> scorers[scorers/base.py]
harness --> trace_utils[trace_utils.py]
harness --> session_utils[session_utils.py]
optimize[optimize.py] --> harness
optimize --> utils[utils.py]
optimize --> prompt_utils[prompt_utils.py]
judges[make_judge.py] --> base[judges/base.py]
judges --> instructions[instructions_judge.py]
scorers[base.py] --> pydantic[pydantic]
scorers --> mlflow_entities[mlflow.entities]
datasets[evaluation_dataset.py] --> mlflow_data[mlflow.data]
datasets --> databricks[DatabricksEvaluationDatasetSource]
examples[evaluate_with_llm_judge.py] --> harness
examples --> judges
examples --> metrics[metrics/genai]
```

**Diagram sources**
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [optimize.py](file://mlflow/genai/optimize/optimize.py)
- [make_judge.py](file://mlflow/genai/judges/make_judge.py)
- [base.py](file://mlflow/genai/scorers/base.py)
- [evaluation_dataset.py](file://mlflow/genai/datasets/evaluation_dataset.py)

**Section sources**
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [optimize.py](file://mlflow/genai/optimize/optimize.py)
- [make_judge.py](file://mlflow/genai/judges/make_judge.py)
- [base.py](file://mlflow/genai/scorers/base.py)
- [evaluation_dataset.py](file://mlflow/genai/datasets/evaluation_dataset.py)

## Performance Considerations
The prompt testing system in MLflow is designed with performance in mind, leveraging parallel execution and efficient resource management. The evaluation harness uses thread pools to process multiple evaluation items concurrently, maximizing throughput while respecting system resource constraints. The system includes configurable parameters for controlling the number of workers, allowing users to balance performance with resource usage.

For large datasets, the system provides streaming capabilities and memory-efficient processing to minimize memory footprint. The optimization framework includes caching mechanisms to avoid redundant computations, and the scoring system supports batch processing for improved efficiency. When working with external LLM APIs, the system implements rate limiting and retry mechanisms to handle API constraints gracefully.

## Troubleshooting Guide
When working with the prompt testing system in MLflow, several common issues may arise. For dataset-related problems, ensure that the dataset contains the required columns (inputs, outputs, expectations) and that the data types are correct. When using traces as input, verify that the traces include span data by using `include_spans=True` in `search_traces()`.

For evaluation failures, check that the predict function is correctly implemented and that it uses the prompt registry appropriately. When using custom scorers, ensure that they return valid types (int, float, bool, str, or Feedback objects). For optimization issues, verify that the prompt URIs are correct and that the target prompts are registered in the prompt registry.

When encountering LLM judge errors, check that the model URI is valid and that the necessary API keys are configured. For performance issues, consider adjusting the worker counts using the MLFLOW_GENAI_EVAL_MAX_WORKERS environment variable.

**Section sources**
- [harness.py](file://mlflow/genai/evaluation/harness.py)
- [optimize.py](file://mlflow/genai/optimize/optimize.py)
- [make_judge.py](file://mlflow/genai/judges/make_judge.py)
- [base.py](file://mlflow/genai/scorers/base.py)
- [utils.py](file://mlflow/genai/evaluation/utils.py)

## Conclusion
The prompt testing system in MLflow provides a comprehensive framework for systematic evaluation and optimization of prompts. By combining automated testing, LLM judges, and optimization algorithms, the system enables developers to rigorously assess and improve prompt quality. The modular architecture supports extensibility while maintaining integration with MLflow's tracking and logging capabilities. With support for various data formats, scoring mechanisms, and optimization strategies, the system offers a flexible solution for prompt testing needs. The implementation balances performance, usability, and robustness, making it suitable for both beginners and experienced developers working with generative AI applications.