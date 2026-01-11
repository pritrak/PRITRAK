# 🚀 PRITRAK Quick Start Guide

**Get PRITRAK running in 5 minutes!**

---

## 1. Installation (1 min)

```bash
# Clone repository
git clone https://github.com/pritrak/PRITRAK.git
cd PRITRAK

# Install dependencies
pip install -r requirements.txt
```

**That's it!** You're ready to classify files.

---

## 2. Classify Your First File (30 seconds)

### Option A: Python API (Recommended)

```python
from pritrak import DataClassifier

# Initialize
classifier = DataClassifier()

# Classify a file
result = classifier.classify_file('path/to/your_document.xlsx')

# View result
print(f"Classification: {result['classification']}")
print(f"Confidence: {result['confidence']}%")
print(f"Risk Level: {result['risk_level']}")
```

### Option B: Command Line

```bash
python -m pritrak classify path/to/document.xlsx
```

### Output

```json
{
  "file": "your_document.xlsx",
  "classification": "CONFIDENTIAL",
  "risk_level": "HIGH",
  "confidence": 87.3,
  "detected_keywords": ["customer", "financial", "budget"],
  "justification": "Document contains customer and financial data",
  "recommendations": [
    "Apply encryption",
    "Restrict access to authorized personnel",
    "Enable audit logging"
  ]
}
```

---

## 3. Understanding Classifications

### 🟢 PUBLIC (Risk: LOW) ✅ ALLOWED
- Empty files
- Marketing materials
- Public documentation

### 🔵 INTERNAL (Risk: MEDIUM) ✅ ALLOWED + LOGGING
- Internal memos
- System logs
- Generic policies

### 🟠 CONFIDENTIAL (Risk: HIGH) ⚠️ WARN USER
- Customer databases
- Financial reports
- Employee records
- Strategic plans

### 🔴 RESTRICTED (Risk: CRITICAL) 🚫 BLOCKED
- API keys
- Passwords
- Private keys
- Credit card numbers
- SSN/PII
- Trade secrets

---

## 4. Batch Classification (2 min)

### Classify Multiple Files

```python
from pritrak import DataClassifier

classifier = DataClassifier()

# Classify all files in a directory
results = classifier.classify_batch('documents/', recursive=True)

# Print summary
print(f"Total files: {len(results)}")

for classification_level in ['PUBLIC', 'INTERNAL', 'CONFIDENTIAL', 'RESTRICTED']:
    count = sum(1 for r in results if r['classification'] == classification_level)
    print(f"{classification_level}: {count} files")

# Show restricted files
restricted = [r for r in results if r['classification'] == 'RESTRICTED']
if restricted:
    print(f"\n🚫 ALERT: {len(restricted)} restricted files found:")
    for r in restricted:
        print(f"  - {r['file']}: {r['justification']}")
```

---

## 5. DLP Integration Example (2 min)

### Prevent Data Loss

```python
from pritrak import DataClassifier

classifier = DataClassifier()

def check_file_transfer(file_path, destination):
    """
    Check if file can be transferred to destination.
    Example destinations: 'external_drive', 'cloud_storage', 'email', 'usb'
    """
    result = classifier.classify_file(file_path)
    
    # Block RESTRICTED classification
    if result['classification'] == 'RESTRICTED':
        return {
            'status': 'BLOCKED',
            'reason': 'File contains restricted data (credentials, PII, trade secrets)',
            'details': result['justification']
        }
    
    # Warn on CONFIDENTIAL to external destinations
    if result['classification'] == 'CONFIDENTIAL':
        if destination in ['external_drive', 'usb', 'email', 'cloud_storage']:
            return {
                'status': 'REQUIRES_APPROVAL',
                'reason': 'Confidential file to external destination',
                'confidence': result['confidence']
            }
    
    # Allow everything else
    return {'status': 'ALLOWED'}

# Usage
check = check_file_transfer('salary_report.xlsx', 'external_drive')
print(check)  # {'status': 'REQUIRES_APPROVAL', 'reason': '...', ...}
```

---

## 6. Real-World Scenarios

### Scenario 1: Block Accidental Credential Upload

```python
# User tries to upload production.env
result = classifier.classify_file('config/production.env')

if result['classification'] == 'RESTRICTED':
    print("🚫 BLOCKED: This file contains API keys and secrets!")
    print(f"Detected: {', '.join(result['detected_patterns'])}")
    # Automatically revoke keys
```

**Result**: API key exposure prevented ✅

### Scenario 2: Classify Customer Data Export

```python
result = classifier.classify_file('exports/customer_database_2026.xlsx')

if result['classification'] == 'CONFIDENTIAL':
    print(f"This file contains customer PII")
    print(f"Confidence: {result['confidence']}%")
    print(f"Required action: Apply encryption, limit access")
```

**Result**: High-value data protected ✅

### Scenario 3: Monitor Directory for Sensitive Files

```python
from pritrak import DataClassifier
import os

classifier = DataClassifier()
sensitive_files = []

for filename in os.listdir('documents/'):
    result = classifier.classify_file(f'documents/{filename}')
    
    if result['classification'] in ['CONFIDENTIAL', 'RESTRICTED']:
        sensitive_files.append(result)

print(f"Found {len(sensitive_files)} sensitive files")
for f in sensitive_files:
    print(f"  - {f['file']}: {f['classification']}")
```

---

## 7. Common Questions

### Q: How accurate is PRITRAK?

**A:** >95% for RESTRICTED detection, >85% for CONFIDENTIAL, <5% false positives. See [Performance Metrics](README.md#performance-metrics) for details.

### Q: Does PRITRAK store file content?

**A:** No. PRITRAK only stores the classification result and metadata. Original file content is never stored or transmitted.

### Q: Can I use it with French documents?

**A:** Yes! PRITRAK automatically detects language (English/French) and uses appropriate keyword lists.

### Q: How fast is it?

**A:** < 100ms for small files, < 5 seconds for 50MB files. Large files use streaming analysis.

### Q: Can I customize the classification?

**A:** Yes. Edit `config/classification_config.json` to adjust thresholds, keywords, and patterns.

### Q: How do I integrate it with my DLP system?

**A:** See [IMPLEMENTATION_GUIDE.md](docs/IMPLEMENTATION_GUIDE.md) for integration examples with popular DLP, SIEM, and cloud platforms.

---

## 8. Next Steps

- 📋 [Read Full Classification Engine Documentation](docs/CLASSIFICATION_ENGINE.md)
- 🛠️ [Explore Implementation Guide](docs/IMPLEMENTATION_GUIDE.md)
- 🔧 [Customize Configuration](config/classification_config.json)
- 📧 [Open GitHub Issues](https://github.com/pritrak/PRITRAK/issues)

---

## 9. Troubleshooting

### Problem: FileNotFoundError

```python
# Make sure file path is correct
result = classifier.classify_file('documents/report.xlsx')  # Correct
result = classifier.classify_file('report.xlsx')            # May fail
```

### Problem: Too many false positives

```python
# Enable false positive reduction
classifier.set_false_positive_reduction(True)

# Or adjust confidence threshold
classifier.set_threshold('confidential', 6.0)  # Higher = fewer FP
```

### Problem: French documents misclassified

```python
# Explicitly set language
result = classifier.classify_file(
    'document.docx',
    language='fr'  # Force French analysis
)
```

---

## 10. Key Files to Know

| File | Purpose |
|------|----------|
| `config/classification_config.json` | Configuration & keywords |
| `docs/CLASSIFICATION_ENGINE.md` | Algorithm documentation |
| `docs/IMPLEMENTATION_GUIDE.md` | Usage & deployment |
| `README.md` | Full documentation |

---

## Support

🔗 **GitHub**: [github.com/pritrak/PRITRAK](https://github.com/pritrak/PRITRAK)  
📧 **Email**: [pritrak47@gmail.com](mailto:pritrak47@gmail.com)  
📚 **Wiki**: [GitHub Wiki](https://github.com/pritrak/PRITRAK/wiki)  

---

**Congratulations!** You're now ready to classify files with PRITRAK. 🚀

**Next**: Try classifying a file from your own documents folder!

```bash
python -m pritrak classify your_document.xlsx
```
