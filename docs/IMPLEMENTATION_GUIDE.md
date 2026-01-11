# PRITRAK Implementation & Usage Guide

**Version:** 1.0  
**Last Updated:** January 11, 2026  
**Purpose:** Step-by-step guide for integrating and deploying the PRITRAK classification engine

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [API Usage](#api-usage)
5. [Real-World Examples](#real-world-examples)
6. [Deployment](#deployment)
7. [Monitoring & Tuning](#monitoring--tuning)
8. [Troubleshooting](#troubleshooting)

---

## Quick Start

### Basic Classification

```python
from pritrak import DataClassifier

# Initialize classifier
classifier = DataClassifier(config_file='config/classification_config.json')

# Classify a file
result = classifier.classify_file('path/to/document.xlsx')

print(f"Classification: {result['classification']}")
print(f"Risk Level: {result['risk_level']}")
print(f"Confidence: {result['confidence']}%")
print(f"Justification: {result['justification']}")
```

### Output Example

```json
{
  "file": "financial_report_2026.xlsx",
  "classification": "CONFIDENTIAL",
  "risk_level": "HIGH",
  "confidence": 87.3,
  "detected_keywords": ["invoice", "budget", "financial forecast"],
  "detected_patterns": ["currency_amounts", "numerical_data"],
  "justification": "Document contains multiple financial keywords and numerical data formatted as tables",
  "recommendations": [
    "Apply encryption to file at rest",
    "Restrict access to authorized personnel only",
    "Enable audit logging for access"
  ],
  "timestamp": "2026-01-11T04:42:12Z"
}
```

---

## Installation

### Prerequisites

- Python 3.8+
- pip or conda
- Optional: Docker for containerized deployment

### From GitHub

```bash
# Clone the repository
git clone https://github.com/pritrak/PRITRAK.git
cd PRITRAK

# Install dependencies
pip install -r requirements.txt

# Or with Poetry
poetry install
```

### Dependencies

```txt
requests>=2.28.0
nltk>=3.8
langdetect>=1.0.9
python-magic>=0.4.24
PyYAML>=6.0
pydantic>=1.9.0
python-dotenv>=0.20.0
```

---

## Configuration

### Using the Configuration File

```python
from pritrak import DataClassifier

# Load from JSON config
classifier = DataClassifier(
    config_file='config/classification_config.json',
    language='en'  # or 'fr' for French
)
```

### Configuration Parameters

```json
{
  "classification_config": {
    "enabled": true,
    "languages": ["en", "fr"],
    "threshold_scores": {
      "restricted": 8.5,
      "confidential": 5.5,
      "internal_medium": 2.0,
      "internal_low": 0.5
    }
  }
}
```

### Environment Variables

```bash
export PRITRAK_CONFIG=/path/to/config.json
export PRITRAK_LOG_LEVEL=INFO
export PRITRAK_CACHE_ENABLED=true
export PRITRAK_MAX_FILE_SIZE=52428800
```

---

## API Usage

### Classification Methods

#### 1. Classify Single File

```python
result = classifier.classify_file(
    file_path='documents/report.pdf',
    detailed=True
)
```

#### 2. Classify File Content

```python
result = classifier.classify_content(
    content=file_content,
    file_name='document.docx',
    language='en'
)
```

#### 3. Batch Classification

```python
files = [
    'documents/file1.xlsx',
    'documents/file2.pdf',
    'documents/file3.docx'
]

results = classifier.classify_batch(files)

for result in results:
    print(f"{result['file']}: {result['classification']}")
```

#### 4. Real-time Stream Classification

```python
# For monitoring uploads/file transfers
stream = classifier.stream_classify(
    directory='watched_folder',
    pattern='*.docx',
    callback=on_file_classified
)

def on_file_classified(result):
    if result['classification'] == 'RESTRICTED':
        print(f"Alert: {result['file']} is {result['classification']}")
```

---

## Real-World Examples

### Example 1: Classify Customer Database Export

```python
result = classifier.classify_file('exports/customer_db_2026-01.xlsx')

if result['classification'] in ['CONFIDENTIAL', 'RESTRICTED']:
    print(f"Blocking transfer: {result['justification']}")
    print(f"Recommended action: {result['recommendations']}")
```

**Expected Output:**
```
Classification: CONFIDENTIAL
Risk: HIGH
Confidence: 86%
Justification: File contains customer contact information and purchase history
Recommendations: ['Apply encryption', 'Restrict to authorized personnel']
```

### Example 2: Detect API Keys in Configuration Files

```python
result = classifier.classify_file('config/production.env')

if result['detected_patterns']:
    print(f"Found patterns: {result['detected_patterns']}")
    # Automatically revoke keys
    for pattern in result['detected_patterns']:
        if pattern.startswith('api_key'):
            print(f"ACTION REQUIRED: Revoke {pattern} immediately")
```

### Example 3: Monitor File Uploads for Data Loss Prevention

```python
from pritrak import DataClassifier

classifier = DataClassifier(config_file='config/classification_config.json')

def check_upload(file_path, destination):
    result = classifier.classify_file(file_path)
    
    # Block RESTRICTED classification
    if result['classification'] == 'RESTRICTED':
        return {
            'status': 'BLOCKED',
            'reason': 'Contains restricted data (trade secrets, credentials, PII)',
            'details': result['justification']
        }
    
    # Warn on CONFIDENTIAL classification
    elif result['classification'] == 'CONFIDENTIAL':
        if destination == 'external_drive' or destination.startswith('http'):
            return {
                'status': 'REQUIRES_APPROVAL',
                'reason': 'Confidential data to external destination',
                'confidence': result['confidence']
            }
    
    return {'status': 'ALLOWED'}

# Usage
upload_result = check_upload('documents/salary_report.xlsx', 's3://bucket')
print(upload_result)
```

### Example 4: Generate Classification Report

```python
import json
from datetime import datetime

files = classifier.classify_batch(
    directory='documents/',
    recursive=True
)

report = {
    'timestamp': datetime.now().isoformat(),
    'total_files': len(files),
    'summary': {
        'PUBLIC': sum(1 for f in files if f['classification'] == 'PUBLIC'),
        'INTERNAL': sum(1 for f in files if f['classification'] == 'INTERNAL'),
        'CONFIDENTIAL': sum(1 for f in files if f['classification'] == 'CONFIDENTIAL'),
        'RESTRICTED': sum(1 for f in files if f['classification'] == 'RESTRICTED')
    },
    'high_risk_files': [f for f in files if f['classification'] in ['CONFIDENTIAL', 'RESTRICTED']],
    'average_confidence': sum(f['confidence'] for f in files) / len(files)
}

with open('classification_report.json', 'w') as f:
    json.dump(report, f, indent=2)
```

---

## Deployment

### Docker Deployment

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PRITRAK_CONFIG=/app/config/classification_config.json

CMD ["python", "-m", "pritrak.server"]
```

```bash
# Build and run
docker build -t pritrak:latest .
docker run -v /path/to/files:/data pritrak:latest
```

### Cloud Deployment (AWS Lambda)

```python
import json
from pritrak import DataClassifier

classifier = DataClassifier(config_file='/opt/config.json')

def lambda_handler(event, context):
    """
    AWS Lambda handler for file classification
    """
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Classify the S3 object
    result = classifier.classify_s3_object(bucket, key)
    
    # Store result
    store_classification_result(result)
    
    # Take action if RESTRICTED
    if result['classification'] == 'RESTRICTED':
        quarantine_file(bucket, key)
        send_alert(result)
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }
```

### Enterprise Integration (SIEM/SOC)

```python
# Integration with Elastic Stack
from pritrak import DataClassifier
from elasticsearch import Elasticsearch

classifier = DataClassifier()
es = Elasticsearch(['localhost:9200'])

def classify_and_index(file_path):
    result = classifier.classify_file(file_path)
    
    # Index in Elasticsearch for SOC monitoring
    es.index(
        index='pritrak-classifications',
        body={
            'timestamp': result['timestamp'],
            'file': result['file'],
            'classification': result['classification'],
            'risk_level': result['risk_level'],
            'confidence': result['confidence'],
            'keywords_detected': result['detected_keywords']
        }
    )
    
    return result
```

---

## Monitoring & Tuning

### Performance Metrics

```python
from pritrak import ClassificationMetrics

metrics = ClassificationMetrics()

# After running classifications
print(f"Average processing time: {metrics.avg_processing_time}ms")
print(f"False positive rate: {metrics.false_positive_rate}%")
print(f"False negative rate: {metrics.false_negative_rate}%")
print(f"Accuracy: {metrics.accuracy}%")
```

### Updating Keywords

```bash
# Update keyword lists quarterly with new threat intelligence
python scripts/update_keywords.py --source=threat-intel-feed
```

### Threshold Tuning

```python
# Adjust confidence thresholds based on your tolerance
classifier.set_threshold('confidential', 6.0)  # Higher = fewer false positives
classifier.set_threshold('restricted', 9.0)
```

---

## Troubleshooting

### Issue: False Positives on Documentation

**Problem:** Markdown files with security examples classified as RESTRICTED

**Solution:**
```python
# Enable false positive reduction
classifier.set_false_positive_reduction(True)
classifier.set_context_awareness('documentation', True)
```

### Issue: Large Files Timeout

**Problem:** Files > 100MB are not classified

**Solution:**
```python
# Enable streaming analysis
classifier.set_streaming_mode(True)
classifier.set_sample_size_mb(5)  # Scan first 5MB
```

### Issue: Multi-Language Detection Fails

**Problem:** French documents classified with English keywords

**Solution:**
```python
result = classifier.classify_file(
    'document.docx',
    force_language='fr'  # Explicitly set language
)
```

---

## Best Practices

1. **Regular Updates**: Update keyword lists monthly with new threat intelligence
2. **Threshold Testing**: A/B test confidence thresholds quarterly
3. **False Positive Audits**: Review misclassified files weekly
4. **Caching**: Enable caching for identical files to improve performance
5. **Monitoring**: Log all classifications for audit trails
6. **Performance**: Monitor average processing time and adjust batch sizes
7. **Integration**: Integrate with existing DLP, SIEM, and SOC systems

---

**For Support:** [GitHub Issues](https://github.com/pritrak/PRITRAK/issues)  
**Documentation:** [Full Documentation](https://github.com/pritrak/PRITRAK/docs)
