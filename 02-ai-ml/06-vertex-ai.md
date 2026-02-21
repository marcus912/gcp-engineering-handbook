# Chapter 6: Vertex AI & Machine Learning on GCP

## 5.1 ML on GCP Overview

Google Cloud offers a comprehensive ML platform:

```
┌─────────────────────────────────────────────────────────────────┐
│                    ML on Google Cloud                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      Vertex AI                            │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │   │
│  │  │  AutoML    │ │ Custom     │ │ Generative AI      │   │   │
│  │  │            │ │ Training   │ │ (Gemini, etc.)     │   │   │
│  │  └────────────┘ └────────────┘ └────────────────────┘   │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │   │
│  │  │ Pipelines  │ │ Model      │ │ Feature Store      │   │   │
│  │  │            │ │ Registry   │ │                    │   │   │
│  │  └────────────┘ └────────────┘ └────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               Pre-trained APIs                            │   │
│  │  Vision AI │ Natural Language │ Speech │ Translation     │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    BigQuery ML                            │   │
│  │             ML directly in SQL                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use Each

| Solution | Use Case | ML Expertise Needed |
|----------|----------|---------------------|
| **Pre-trained APIs** | Common tasks (vision, language) | None |
| **AutoML** | Custom models, no coding | Low |
| **Custom Training** | Full control, complex models | High |
| **BigQuery ML** | ML for analysts, SQL-based | Low |
| **Vertex AI Generative** | LLMs, text/image generation | Low-Medium |

---

## 5.2 Vertex AI Platform

Vertex AI is the unified ML platform that brings together all ML services.

### Key Components

**Data Preparation**:
- Datasets (managed datasets for training)
- Feature Store (feature management)
- Labeling tasks (data annotation)

**Model Development**:
- Workbench (managed Jupyter notebooks)
- Training (custom and AutoML)
- Experiments (track experiments)

**Model Management**:
- Model Registry (version models)
- Endpoints (deploy models)
- Batch Prediction

**MLOps**:
- Pipelines (orchestrate ML workflows)
- Metadata (track lineage)
- Model Monitoring

### Vertex AI Workbench

Managed Jupyter notebook environments:

```bash
# Create a managed notebook instance
gcloud notebooks instances create my-notebook \
  --location=us-central1-a \
  --machine-type=n1-standard-4 \
  --vm-image-project=deeplearning-platform-release \
  --vm-image-family=tf-latest-gpu
```

**Instance types**:
- **Managed notebooks**: Fully managed, auto-upgrades
- **User-managed notebooks**: More control, you manage updates

---

## 5.3 AutoML

Train custom models without writing code.

### AutoML Capabilities

| Type | Task | Input |
|------|------|-------|
| **AutoML Image** | Classification, Object Detection, Segmentation | Images |
| **AutoML Video** | Classification, Object Tracking, Action Recognition | Videos |
| **AutoML Text** | Classification, Entity Extraction, Sentiment | Text |
| **AutoML Tabular** | Classification, Regression, Forecasting | Structured data |

### AutoML Tabular Example

```python
from google.cloud import aiplatform

aiplatform.init(project='my-project', location='us-central1')

# Create dataset
dataset = aiplatform.TabularDataset.create(
    display_name='my-dataset',
    gcs_source='gs://bucket/data.csv'
)

# Train model
job = aiplatform.AutoMLTabularTrainingJob(
    display_name='my-automl-job',
    optimization_prediction_type='classification',
    column_specs={
        'feature1': 'numeric',
        'feature2': 'categorical',
        'target': 'categorical'
    }
)

model = job.run(
    dataset=dataset,
    target_column='target',
    training_fraction_split=0.8,
    validation_fraction_split=0.1,
    test_fraction_split=0.1
)
```

### AutoML Best Practices

1. **Data quality matters**: Clean, representative data
2. **Sufficient data**: Minimum varies by task (e.g., 1000 images for classification)
3. **Balanced classes**: Avoid severe class imbalance
4. **Proper train/test split**: Ensure no data leakage

---

## 5.4 Custom Training

Full control over model architecture and training.

### Training Methods

**Pre-built containers**:
- TensorFlow, PyTorch, XGBoost, Scikit-learn
- Google-maintained, optimized

**Custom containers**:
- Your own Docker image
- Any framework

### Custom Training Job

```python
from google.cloud import aiplatform

aiplatform.init(project='my-project', location='us-central1')

# Create custom training job
job = aiplatform.CustomTrainingJob(
    display_name='my-training-job',
    script_path='trainer/task.py',
    container_uri='us-docker.pkg.dev/vertex-ai/training/tf-cpu.2-12:latest',
    requirements=['pandas', 'scikit-learn'],
    model_serving_container_image_uri='us-docker.pkg.dev/vertex-ai/prediction/tf2-cpu.2-12:latest'
)

# Run training
model = job.run(
    replica_count=1,
    machine_type='n1-standard-4',
    accelerator_type='NVIDIA_TESLA_T4',
    accelerator_count=1,
    args=['--epochs=100', '--batch-size=32']
)
```

### Training Script Structure

```python
# trainer/task.py
import argparse
import os
from google.cloud import storage

def train(args):
    # Load data
    # ...

    # Train model
    # ...

    # Save model to GCS
    model_dir = os.environ.get('AIP_MODEL_DIR', args.model_dir)
    model.save(model_dir)

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--epochs', type=int, default=10)
    parser.add_argument('--batch-size', type=int, default=32)
    parser.add_argument('--model-dir', type=str, default='gs://bucket/model')
    args = parser.parse_args()
    train(args)
```

### Hyperparameter Tuning

```python
from google.cloud import aiplatform

job = aiplatform.HyperparameterTuningJob(
    display_name='hpt-job',
    custom_job=custom_job,
    metric_spec={'accuracy': 'maximize'},
    parameter_spec={
        'learning_rate': aiplatform.hyperparameter_tuning.DoubleParameterSpec(
            min=0.001, max=0.1, scale='log'
        ),
        'batch_size': aiplatform.hyperparameter_tuning.DiscreteParameterSpec(
            values=[16, 32, 64, 128], scale='unspecified'
        )
    },
    max_trial_count=20,
    parallel_trial_count=5
)

job.run()
```

---

## 5.5 Model Deployment

### Deploying to Endpoint

```python
from google.cloud import aiplatform

# Get model
model = aiplatform.Model('projects/PROJECT/locations/REGION/models/MODEL_ID')

# Deploy to endpoint
endpoint = model.deploy(
    deployed_model_display_name='my-deployment',
    machine_type='n1-standard-4',
    min_replica_count=1,
    max_replica_count=10,
    accelerator_type='NVIDIA_TESLA_T4',
    accelerator_count=1
)

# Make prediction
response = endpoint.predict(instances=[
    {'feature1': 1.0, 'feature2': 'value'}
])
```

### Traffic Splitting

Deploy multiple model versions:

```python
endpoint.deploy(
    model=new_model,
    traffic_percentage=10,  # Send 10% to new model
    deployed_model_display_name='canary'
)
```

### Batch Prediction

For large-scale offline predictions:

```python
batch_prediction_job = model.batch_predict(
    job_display_name='batch-job',
    gcs_source='gs://bucket/input.jsonl',
    gcs_destination_prefix='gs://bucket/output/',
    machine_type='n1-standard-4',
    starting_replica_count=2,
    max_replica_count=10
)
```

---

## 5.6 MLOps with Vertex AI

### Vertex AI Pipelines

Orchestrate ML workflows using Kubeflow Pipelines or TFX:

```python
from kfp.v2 import dsl
from kfp.v2.dsl import component
from google.cloud import aiplatform

@component
def preprocess_data(input_data: str, output_data: str):
    # Preprocessing logic
    pass

@component
def train_model(data: str, model_output: str):
    # Training logic
    pass

@component
def deploy_model(model: str):
    # Deployment logic
    pass

@dsl.pipeline(name='my-pipeline')
def ml_pipeline(input_data: str):
    preprocess_task = preprocess_data(input_data=input_data, output_data='gs://bucket/processed')
    train_task = train_model(data=preprocess_task.output, model_output='gs://bucket/model')
    deploy_task = deploy_model(model=train_task.output)

# Compile and run
from kfp.v2 import compiler

compiler.Compiler().compile(
    pipeline_func=ml_pipeline,
    package_path='pipeline.json'
)

aiplatform.PipelineJob(
    display_name='my-pipeline-run',
    template_path='pipeline.json',
    parameter_values={'input_data': 'gs://bucket/raw-data'}
).run()
```

### Feature Store

Centralized feature management:

```python
from google.cloud import aiplatform

# Create feature store
feature_store = aiplatform.Featurestore.create(
    featurestore_id='my-feature-store',
    online_store_fixed_node_count=1
)

# Create entity type
entity_type = feature_store.create_entity_type(
    entity_type_id='users',
    description='User features'
)

# Create features
entity_type.create_feature(
    feature_id='age',
    value_type='INT64'
)

entity_type.create_feature(
    feature_id='purchase_count',
    value_type='INT64'
)

# Ingest features
entity_type.ingest_from_gcs(
    gcs_source_uris='gs://bucket/features.csv',
    gcs_source_type='csv',
    entity_id_field='user_id',
    feature_ids=['age', 'purchase_count']
)

# Online serving
entity_type.read(entity_ids=['user_123', 'user_456'])
```

### Model Monitoring

Detect drift and anomalies:

```python
from google.cloud import aiplatform

# Create monitoring job
aiplatform.ModelDeploymentMonitoringJob.create(
    display_name='my-monitoring',
    endpoint=endpoint,
    logging_sampling_strategy={'random_sample_config': {'sample_rate': 0.8}},
    schedule_config={'monitor_interval': {'seconds': 3600}},
    objective_configs=[{
        'deployed_model_id': deployed_model.id,
        'objective_config': {
            'training_dataset': {
                'gcs_source': {'uris': ['gs://bucket/training-data.csv']},
                'data_format': 'csv'
            },
            'training_prediction_skew_detection_config': {
                'skew_thresholds': {'feature1': {'value': 0.3}}
            }
        }
    }]
)
```

---

## 5.7 Pre-trained APIs

Ready-to-use ML models via API.

### Vision AI

```python
from google.cloud import vision

client = vision.ImageAnnotatorClient()

# Load image
with open('image.jpg', 'rb') as f:
    content = f.read()

image = vision.Image(content=content)

# Label detection
response = client.label_detection(image=image)
for label in response.label_annotations:
    print(f'{label.description}: {label.score}')

# Object detection
response = client.object_localization(image=image)
for obj in response.localized_object_annotations:
    print(f'{obj.name}: {obj.score}')

# OCR
response = client.text_detection(image=image)
print(response.full_text_annotation.text)

# Face detection
response = client.face_detection(image=image)
for face in response.face_annotations:
    print(f'Joy: {face.joy_likelihood}')
```

### Natural Language AI

```python
from google.cloud import language_v1

client = language_v1.LanguageServiceClient()

document = language_v1.Document(
    content="Google Cloud is great for machine learning.",
    type_=language_v1.Document.Type.PLAIN_TEXT
)

# Sentiment analysis
sentiment = client.analyze_sentiment(document=document)
print(f'Score: {sentiment.document_sentiment.score}')
print(f'Magnitude: {sentiment.document_sentiment.magnitude}')

# Entity extraction
entities = client.analyze_entities(document=document)
for entity in entities.entities:
    print(f'{entity.name}: {entity.type_.name}')

# Classification
classification = client.classify_text(document=document)
for category in classification.categories:
    print(f'{category.name}: {category.confidence}')
```

### Speech-to-Text

```python
from google.cloud import speech

client = speech.SpeechClient()

with open('audio.wav', 'rb') as f:
    content = f.read()

audio = speech.RecognitionAudio(content=content)
config = speech.RecognitionConfig(
    encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
    sample_rate_hertz=16000,
    language_code='en-US'
)

response = client.recognize(config=config, audio=audio)
for result in response.results:
    print(result.alternatives[0].transcript)
```

### Translation AI

```python
from google.cloud import translate_v2

client = translate_v2.Client()

# Translate text
result = client.translate('Hello, world!', target_language='zh-TW')
print(result['translatedText'])

# Detect language
detection = client.detect_language('Bonjour le monde!')
print(detection['language'])
```

---

## 5.8 BigQuery ML

Train models using SQL.

### Supported Model Types

| Model Type | Use Case |
|------------|----------|
| LINEAR_REG | Linear regression |
| LOGISTIC_REG | Binary/multi-class classification |
| KMEANS | Clustering |
| BOOSTED_TREE_CLASSIFIER | Classification with XGBoost |
| BOOSTED_TREE_REGRESSOR | Regression with XGBoost |
| DNN_CLASSIFIER | Deep neural network classification |
| DNN_REGRESSOR | Deep neural network regression |
| AUTOML_CLASSIFIER | AutoML classification |
| AUTOML_REGRESSOR | AutoML regression |

### Example: Classification Model

```sql
-- Create training data view
CREATE OR REPLACE VIEW `project.dataset.training_data` AS
SELECT
  feature1,
  feature2,
  feature3,
  label
FROM `project.dataset.raw_data`
WHERE split = 'train';

-- Create model
CREATE OR REPLACE MODEL `project.dataset.my_model`
OPTIONS(
  model_type='LOGISTIC_REG',
  input_label_cols=['label'],
  auto_class_weights=TRUE,
  max_iterations=20
) AS
SELECT * FROM `project.dataset.training_data`;

-- Evaluate model
SELECT * FROM ML.EVALUATE(MODEL `project.dataset.my_model`);

-- Make predictions
SELECT * FROM ML.PREDICT(
  MODEL `project.dataset.my_model`,
  (SELECT feature1, feature2, feature3 FROM `project.dataset.new_data`)
);

-- Explain predictions
SELECT * FROM ML.EXPLAIN_PREDICT(
  MODEL `project.dataset.my_model`,
  (SELECT * FROM `project.dataset.new_data`),
  STRUCT(3 AS top_k_features)
);
```

---

## 5.9 ML Troubleshooting

### Common Issues

**Training job fails**:
1. Check logs in Cloud Logging
2. Verify dataset exists and is accessible
3. Check quota (GPUs, CPUs)
4. Verify container image exists
5. Check code for errors

**Model prediction latency is high**:
1. Check model complexity
2. Consider model optimization (quantization, pruning)
3. Use appropriate machine type
4. Check input preprocessing time
5. Consider batch prediction for large volumes

**Model accuracy degradation**:
1. Check for data drift
2. Verify feature pipeline hasn't changed
3. Check for upstream data quality issues
4. Retrain with recent data

**Deployment fails**:
1. Check model artifact exists in GCS
2. Verify serving container image
3. Check model signature (input/output format)
4. Check quota for endpoint resources

### Debugging Commands

```bash
# List training jobs
gcloud ai custom-jobs list --region=us-central1

# Get job details
gcloud ai custom-jobs describe JOB_ID --region=us-central1

# Stream job logs
gcloud ai custom-jobs stream-logs JOB_ID --region=us-central1

# List endpoints
gcloud ai endpoints list --region=us-central1

# Describe endpoint
gcloud ai endpoints describe ENDPOINT_ID --region=us-central1
```

---

## Chapter 6 Review Questions

1. When would you use AutoML vs custom training?

2. How would you handle a customer whose model accuracy dropped in production?

3. Explain the purpose of Vertex AI Feature Store.

4. What's the difference between online prediction and batch prediction?

5. How would you monitor for model drift?

6. A customer wants to add ML to their existing BigQuery analytics. What do you recommend?

---

## Chapter 6 Hands-On Exercises

### Exercise 5.1: AutoML
1. Upload a sample dataset to Vertex AI
2. Train an AutoML tabular model
3. Evaluate the model
4. Deploy to an endpoint and make predictions

### Exercise 5.2: Vision API
1. Use the Vision API to analyze images
2. Try label detection, OCR, and face detection
3. Build a simple image classification app

### Exercise 5.3: BigQuery ML
1. Create a classification model in BigQuery ML
2. Evaluate the model
3. Make predictions on new data
4. Export the model to Vertex AI

---

## Key Takeaways

1. **Vertex AI is the unified platform** - Use it for most ML tasks
2. **Start with pre-trained APIs** - Don't reinvent the wheel
3. **AutoML for quick custom models** - When pre-trained doesn't fit
4. **Custom training for control** - When you need specific architectures
5. **Feature Store for consistency** - Same features in training and serving
6. **Monitor models in production** - Drift happens

---

[Next Chapter: Generative AI on GCP →](./06-generative-ai.md)
