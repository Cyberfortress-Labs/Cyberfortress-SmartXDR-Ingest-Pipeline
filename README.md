# SmartXDR Ingest Pipeline

Automatic log classification system using Machine Learning on Elasticsearch.

## Architecture

```
Elastic Agent / Filebeat
         │
         ▼
Fleet Pipeline (logs-suricata.eve-2.24.0, logs-zeek.*, ...)
         │
         ▼
logs@custom (entry point)
         │
         ▼
smartxdr-ingest-pipeline (ML Classification)
         │
         ▼
Elasticsearch Index (with ml.prediction)
```


## ML Model: Bylastic Classification Logs

This pipeline uses the **Bylastic** model from Hugging Face for log classification.

- **Model**: [byviz/bylastic_classification_logs](https://huggingface.co/byviz/bylastic_classification_logs)
- **License**: Apache 2.0
- **Created by**: Byviz Analytics

### Key Features

- **Accurate Classification**: Classifies logs into three critical categories: **ERROR**, **WARNING**, and **INFO**
- **Elastic Compatibility**: Designed to seamlessly integrate with Elasticsearch
- **High Performance**: Optimized to process large volumes of logs efficiently
- **Easy Integration**: Can be easily integrated into existing log processing pipelines

### Log Categories

| Category    | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| **ERROR**   | Critical failures or serious problems requiring immediate attention |
| **WARNING** | Potential issues that could become errors if not properly managed   |
| **INFO**    | Informational logs about normal system functioning                  |

### Load Model (Python)

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification

tokenizer = AutoTokenizer.from_pretrained("byviz/bylastic_classification_logs")
model = AutoModelForSequenceClassification.from_pretrained("byviz/bylastic_classification_logs")
```

### Model Performance

Bylastic utilizes advanced NLP techniques and has been trained with diverse log data for high classification accuracy. For detailed performance comparisons (Bylastic vs BERT), visit the [model page on Hugging Face](https://huggingface.co/byviz/bylastic_classification_logs).


### Key Features

- **Accurate Classification**: Classifies logs into three critical categories: **ERROR**, **WARNING**, and **INFO**
- **Elastic Compatibility**: Designed to seamlessly integrate with Elasticsearch
- **High Performance**: Optimized to process large volumes of logs efficiently
- **Easy Integration**: Can be easily integrated into existing log processing pipelines

### Log Categories

| Category    | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| **ERROR**   | Critical failures or serious problems requiring immediate attention |
| **WARNING** | Potential issues that could become errors if not properly managed   |
| **INFO**    | Informational logs about normal system functioning                  |

### Load Model (Python)

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification

tokenizer = AutoTokenizer.from_pretrained("byviz/bylastic_classification_logs")
model = AutoModelForSequenceClassification.from_pretrained("byviz/bylastic_classification_logs")
```

### Model Performance

Bylastic utilizes advanced NLP techniques and has been trained with diverse log data for high classification accuracy. For detailed performance comparisons (Bylastic vs BERT), visit the [model page on Hugging Face](https://huggingface.co/byviz/bylastic_classification_logs).


## Installation

### 1. Deploy ML Model

```bash
# Upload model to Elasticsearch (do this first)
# Model ID: byviz__bylastic_classification_logs
```

### 2. Create ML Classification Pipeline

```bash
# PUT _ingest/pipeline/smartxdr-ingest-pipeline
curl -X PUT "https://<ES_HOST>:9200/_ingest/pipeline/smartxdr-ingest-pipeline" \
  -H "Content-Type: application/json" \
  -u "<username>:<password>" \
  -d @pipelines/smartxdr-ingest-pipeline.json
```

### 3. Create Entry Point Pipeline

```bash
curl -X PUT "https://<ES_HOST>:9200/_ingest/pipeline/logs@custom" \
  -H "Content-Type: application/json" \
  -u "<username>:<password>" \
  -d '{
    "description": "Custom pipeline entry point - calls SmartXDR",
    "processors": [{
      "pipeline": {
        "name": "smartxdr-ingest-pipeline",
        "ignore_failure": true
      }
    }]
  }'
```

## Directory Structure

```
elasticsearch_ingest/
├── README.md                              # This document
├── pipelines/
│   └── smartxdr-ingest-pipeline.json      # Pipeline definition
├── scripts/
│   └── smartxdr-log-classifier.painless   # Painless script
└── config-model.http                      # Sample HTTP requests
```

## Supported Log Sources

| Source          | Classification Condition                                                               |
| --------------- | -------------------------------------------------------------------------------------- |
| **Suricata**    | `event_type == "alert"`                                                                |
| **Zeek**        | `event.kind == "alert"` and has `zeek.notice`                                          |
| **Wazuh**       | Has `rule.description`                                                                 |
| **pfSense**     | `action == "block"` or `"reject"`                                                      |
| **Apache**      | Level `error/crit/alert/emerg/warning` or message contains alert keywords              |
| **Nginx**       | Level not `notice`, not startup messages                                               |
| **MySQL**       | Level `Warning`                                                                        |
| **Windows**     | `event.kind == "alert"` or level `warning/error/critical` or contains malware keywords |
| **ModSecurity** | Has `modsec.audit.messages` and `url.query`                                            |

## Output

Classified logs will have additional fields:

```json
{
  "ml_input": "Suricata: ET SCAN ... | Category: Potentially Bad Traffic | 10.0.0.1 -> 192.168.1.1",
  "ml": {
    "prediction": {
      "predicted_value": "WARNING",
      "prediction_probability": 0.85,
      "model_id": "byviz__bylastic_classification_logs"
    }
  }
}
```

### Classification Labels

| Label      | Description                                 |
| ---------- | ------------------------------------------- |
| `INFO`     | Normal information                          |
| `WARNING`  | Alert requiring attention                   |
| `CRITICAL` | Serious incident requiring immediate action |

## Testing

### Simulate Pipeline

```bash
POST _ingest/pipeline/smartxdr-ingest-pipeline/_simulate
{
  "docs": [
    {
      "_index": "logs-suricata.eve-default",
      "_source": {
        "suricata": { "eve": { "event_type": "alert" }},
        "rule": { "name": "ET SCAN Test", "category": "Attempted Scan" },
        "source": { "ip": "10.0.0.1" },
        "destination": { "ip": "192.168.1.1" }
      }
    }
  ]
}
```

### Query Classified Logs

```bash
GET logs-suricata.eve-*/_search
{
  "query": { "exists": { "field": "ml.prediction" }},
  "size": 10,
  "sort": [{ "@timestamp": "desc" }]
}
```

## Important Notes

1. **Do not modify Fleet pipelines** - Use `logs@custom` to hook into the pipeline chain
2. **Model must be deployed** - Verify ML model is running:
   ```bash
   GET _ml/trained_models/byviz__bylastic_classification_logs/_stats
   ```
3. **Existing logs are not classified** - Pipeline only applies to new logs

## Troubleshooting

### Pipeline Not Running

```bash
# Check pipeline exists
GET _ingest/pipeline/logs@custom
GET _ingest/pipeline/smartxdr-ingest-pipeline

# Check for error tags in logs
GET logs-*/_search
{
  "query": { "term": { "tags": "_ml_inference_failure" }}
}
```

### Model Not Working

```bash
# Check model stats
GET _ml/trained_models/byviz__bylastic_classification_logs/_stats

# Deploy model if not running
POST _ml/trained_models/byviz__bylastic_classification_logs/deployment/_start
```
