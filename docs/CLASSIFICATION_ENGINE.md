# 🔐 PRITRAK Advanced Data Classification Engine

**Version:** 1.0 (Production-Ready)  
**Based On:** NIST SP 800-60/800-53, ISO 27001, Real-World DLP Standards  
**Target:** English & French Documents in Enterprise/Corporate Environments  
**Focus:** Prevent data loss while minimizing false positives  

---

## 📊 CLASSIFICATION MATRIX

### 1. **PUBLIC** - No Risk
Files that pose NO security risk if exposed externally

#### Characteristics:
- Empty files (0 bytes)
- Plain whitespace/formatting only
- System/temporary files
- Marketing materials, public documents
- File naming suggests public nature

#### Detection Rules:
```
FILE SIZE = 0 BYTES → PUBLIC (Confidence: 100%)
FILE SIZE < 100 BYTES AND ONLY WHITESPACE → PUBLIC (Confidence: 95%)
EXTENSION IN [.txt, .md, .csv] AND NO SENSITIVE KEYWORDS → Evaluate further
KEYWORDS: "public", "readme", "license", "marketing", "brochure", "whitepaper" (non-confidential) → PUBLIC (Confidence: 80%)
```

#### Risk Level: **LOW**

---

### 2. **INTERNAL** - Low Risk
Information for internal use only, limited disruption if exposed

#### English Keywords:
- Internal IP/network: `192.168.`, `10.0.`, `172.16.`, `IntranetURL`, `internal-server`
- Internal communications: "internal memo", "internal note", "staff update", "team announcement"
- Internal policies: "internal policy", "procedure guide", "workplace guidelines" (non-sensitive)
- Internal project names: "Project Q2", "Initiative 2026" (generic)
- System logs: server logs, application logs (non-sensitive)

#### French Keywords:
- `192.168.`, `10.0.`, `172.16.` (IPs remain same)
- "mémo interne", "note interne", "communication interne", "mise à jour du personnel"
- "politique interne", "guide de procédure", "directives de travail"
- "Projet", "Initiative", "Mise à jour d'équipe"
- "Journal système", "journal serveur"

#### Regex Patterns - Internal IP Detection:
```
IPv4 PRIVATE: ^(192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[01])\.)
IPv6 PRIVATE: ^(fd|fe80:|::1)
INTERNAL DOMAIN: \b(intranet|internal|local|corp)\..+
```

#### Document Indicators:
- No external email domains
- No PII, financial data, or trade secrets
- Department-level communications
- Employee handbooks (generic, non-personal)
- Meeting minutes (non-sensitive topics)

#### Risk Level: **LOW**

---

### 3. **CONFIDENTIAL** - Medium-High Risk
Restricted to specific groups; unauthorized access = business disruption

#### English Keywords (HIGH CONFIDENCE):
- **Financial Data:** "invoice", "balance sheet", "P&L", "payroll", "expense report", "salary", "budget", "forecast", "financial statement"
- **Customer Data:** "customer", "client", "account number", "order ID", "customer list", "customer database"
- **Employee Data:** "employee record", "personnel file", "performance review", "employee ID", "hire date", "medical leave"
- **Business Plans:** "strategic plan", "business strategy", "product roadmap", "market analysis", "competitive analysis"
- **Contracts:** "agreement", "contract", "terms and conditions", "NDA", "confidential agreement"
- **Source Code:** "source code", "API key", "database password", "private repository" (code files)
- **Technical Architecture:** "architecture diagram", "system design", "database schema", "network topology"
- **Meeting Notes:** "confidential", "sensitive topic", "executive discussion"

#### French Keywords (HIGH CONFIDENCE):
- **Financial Data:** "facture", "bilan", "compte de résultat", "paie", "rapport de dépenses", "salaire", "budget", "prévision", "état financier"
- **Customer Data:** "client", "numéro de compte", "identifiant de commande", "liste des clients", "base de données client"
- **Employee Data:** "fiche de personnel", "dossier de personnel", "examen de performance", "identifiant employé", "date d'embauche", "congé maladie"
- **Business Plans:** "plan stratégique", "stratégie commerciale", "feuille de route produit", "analyse du marché", "analyse concurrentielle"
- **Contracts:** "accord", "contrat", "conditions générales", "accord de confidentialité", "NDA"
- **Source Code:** "code source", "clé API", "mot de passe base de données", "référentiel privé"
- **Technical Architecture:** "diagramme d'architecture", "conception système", "schéma de base de données", "topologie réseau"

#### Regex Patterns - CONFIDENTIAL PII Detection:
```
CREDIT CARD: \b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13}|3(?:0[0-5]|[68][0-9])[0-9]{11}|6(?:011|5[0-9]{2})[0-9]{12}|(?:2131|1800|35\d{3})\d{11})\b

US SSN: \b(?!000|666|9\d{2})([0-8]\d{2}|7([0-6]\d|7[012]))([-]?)\d{2}\3\d{4}\b

BANK ACCOUNT: \b\d{8,17}\b (with context check)

IBAN: \b[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}\b

FRENCH SECU: \b[1-9]\d{2}\s?[0-9]{2}\s?[0-9]{2}\s?[0-9]{3}\s?[0-9]{3}\s?[0-9]{2}\d\b

PASSPORT: \b[A-Z]{1,2}[0-9]{6,9}\b
```

#### Document Type Indicators:
- Spreadsheets with customer/financial data
- HR personnel files (even generic)
- Product/service contract documents
- Meeting minutes with business-sensitive content
- Internal financial reports
- System/network diagrams

#### Risk Level: **MEDIUM to HIGH**

---

### 4. **RESTRICTED** - CRITICAL Risk
Trade secrets, highest regulatory impact; severe financial/legal consequences if exposed

#### English Keywords (CRITICAL CONFIDENCE):
- **Trade Secrets:** "trade secret", "proprietary", "patent", "R&D", "research and development", "confidential technology"
- **Credentials & Keys:** "password", "API key", "private key", "secret key", "access token", "authentication", "encryption key"
- **Security:** "vulnerability", "exploit", "zero-day", "security flaw", "backdoor", "penetration test", "security audit"
- **Regulated PII:** "SSN", "social security", "passport", "driver's license", "medical record", "health information", "diagnosis"
- **Financial Instruments:** "credit card", "bank account", "SWIFT", "routing number", "treasury", "hedge fund"
- **Executive/Board Info:** "board meeting", "executive session", "board resolution", "confidential to board"
- **Mergers & Acquisitions:** "acquisition", "merger", "asset sale", "due diligence", "deal", "valuation"
- **Regulatory/Legal:** "regulatory investigation", "compliance violation", "legal action", "litigation", "subpoena"

#### French Keywords (CRITICAL CONFIDENCE):
- **Trade Secrets:** "secret commercial", "propriétaire", "brevet", "recherche et développement", "technologie confidentielle"
- **Credentials & Keys:** "mot de passe", "clé API", "clé privée", "clé secrète", "jeton d'accès", "authentification", "clé de chiffrement"
- **Security:** "vulnérabilité", "exploit", "zéro-jour", "faille de sécurité", "porte dérobée", "test de pénétration", "audit de sécurité"
- **Regulated PII:** "Numéro de sécurité sociale", "passeport", "permis de conduire", "dossier médical", "information de santé", "diagnostic"
- **Financial Instruments:** "carte de crédit", "compte bancaire", "SWIFT", "numéro de routage", "trésorier", "fonds spéculatif"
- **Executive/Board Info:** "réunion du conseil", "session exécutive", "résolution du conseil"
- **Mergers & Acquisitions:** "acquisition", "fusion", "vente d'actifs", "due diligence", "accord", "évaluation"
- **Regulatory/Legal:** "enquête réglementaire", "violation de conformité", "action en justice", "litige", "assignation"

#### Regex Patterns - RESTRICTED Data Detection:
```
DATABASE DUMP: (?i)(mysqldump|postgresql|oracle|sqlserver).*\.sql|table.*\(.*\).*values

PRIVATE KEY (PEM/DER): -----BEGIN (RSA|DSA|EC|OPENSSH|ENCRYPTED) PRIVATE KEY-----

API KEY FORMATS:
  AWS: AKIA[0-9A-Z]{16}
  GCP: AIza[0-9A-Za-z_\-]{35}
  GitHub: ghp_[0-9a-zA-Z]{36}
  Slack: xox[baprs]-[0-9]{10,13}-[a-zA-Z0-9_\-]{24,34}

ENCRYPTION KEY: (?i)(aes|rsa|256|sha|hmac|cipher).*?key.*?=.*?[0-9a-f]{32,}

DATABASE CONNECTION STRING: (?i)database.*?=.*?password.*?=|server=.*?user.*?pass

JWT TOKEN: eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+

PHONE (International): \+?[1-9]\d{1,14}

MEDICAL/HIPAA: (?i)(ICD-10|procedure code|diagnosis|treatment plan|medication|prescription)

FRENCH EMPLOYEE ID: \b[0-9]{1,3}\s[0-9]{1,3}\s[0-9]{1,3}\b
```

#### File Type Indicators:
- `.key`, `.pem`, `.p12`, `.pfx` (encryption keys)
- `.db`, `.mdb` (database files with credentials)
- `.sql` (database dumps)
- Password managers exports
- Configuration files with API keys
- Firmware/binary files (potential exploits)
- Patent documents
- Board/executive presentations

#### Risk Level: **CRITICAL**

---

## 🎯 ADVANCED CLASSIFICATION ALGORITHM

### Phase 1: Pre-Analysis (Lightweight)
```
INPUT: File path, file size, extension

1. SIZE CHECK
   - IF file_size == 0 → CLASSIFICATION = PUBLIC, RISK = LOW
   - IF file_size < 100 AND only_whitespace → CLASSIFICATION = PUBLIC, RISK = LOW
   
2. EXTENSION CHECK (Quick Win)
   - IF extension in [.key, .pem, .p12, .pfx, .db, .sql] 
     AND file_size > 1KB → Assume RESTRICTED, proceed to Phase 2
   - IF extension in [.docx, .xlsx, .pdf] → Proceed to Phase 2
   - IF extension in [.exe, .dll, .sys] → CLASSIFICATION = INTERNAL, RISK = MEDIUM (execution risk)
   
3. FILENAME ANALYSIS
   - IF filename contains "password", "secret", "key", "api" (case-insensitive) 
     → CLASSIFICATION = RESTRICTED, RISK = CRITICAL
   - IF filename contains "employee", "payroll", "salary", "customer" 
     → CLASSIFICATION = CONFIDENTIAL, RISK = HIGH
   - IF filename contains "public", "readme", "license", "marketing" 
     → CLASSIFICATION = PUBLIC, RISK = LOW
   
4. SIZE-BASED HEURISTIC
   - IF file_size > 50MB AND binary → Likely database dump → RESTRICTED
   - IF file_size < 10KB AND text → Likely config/credentials → RESTRICTED (after Phase 2)
```

### Phase 2: Content Analysis (Detailed)

#### Step A: Language Detection
```
SCAN FIRST 500 CHARACTERS:
- IF French_score > 60% → Use FRENCH keyword set
- ELSE → Use ENGLISH keyword set
- IF multilingual → Use BOTH keyword sets
```

#### Step B: Keyword Scanning (Weighted Scoring)
```
confidence_score = 0
max_score = 0

FOR EACH classification_level in [RESTRICTED, CONFIDENTIAL, INTERNAL]:
    FOR EACH keyword in keywords[classification_level]:
        IF keyword found in file_content:
            - Base weight = classification_level.weight (R:3.0, C:2.0, I:1.0)
            - Context multiplier = checkContext(keyword, surrounding_text)
              * If keyword in TITLE/HEADER: multiply by 1.5
              * If keyword in TABLE/DATA: multiply by 1.3
              * If keyword in FILENAME: multiply by 2.0
            - confidence_score += base_weight * context_multiplier
            - Track keyword frequencies

RESULT: weighted_keywords = {keyword: score, count: occurrences}
```

#### Step C: Pattern Detection (Regex Confidence)
```
pattern_detections = {}

FOR EACH data_type in [CREDIT_CARD, SSN, EMAIL, IP_ADDRESS, ...]:
    matches = regex_scan(content, pattern[data_type])
    
    IF matches.count > 0:
        pattern_detections[data_type] = {
            count: matches.count,
            confidence: calculate_confidence(data_type, matches),
            risk_elevation: get_risk_level(data_type)
        }
```

### Phase 3: Final Scoring
```
FINAL_SCORE = {
    RESTRICTED_weighted_keywords_score +
    RESTRICTED_pattern_detections_score * 2.5 +
    structure_analysis_score +
    (entropy_score if encrypted else 0) +
    filename_analysis_score +
    file_context_boost
}

CONFIDENCE = (FINAL_SCORE / MAX_POSSIBLE_SCORE) * 100

Thresholds (tuned):
- FINAL_SCORE >= 8.5 → RESTRICTED (CRITICAL)
- FINAL_SCORE >= 5.5 → CONFIDENTIAL (HIGH)
- FINAL_SCORE >= 2.0 → INTERNAL (MEDIUM)
- FINAL_SCORE >= 0.5 → INTERNAL (LOW)
- FINAL_SCORE < 0.5 → PUBLIC (LOW)
```

---

## 🛡️ EDGE CASES

### Edge Case 1: Empty Files
```
File Size = 0 bytes OR only whitespace
ACTION: ALWAYS → PUBLIC, RISK = LOW, CONFIDENCE = 100%
REASON: No data = no risk
```

### Edge Case 2: Encrypted/Compressed Files
```
IF file extension in [.zip, .rar, .7z, .tar.gz]:
    - Check for "confidential", "secret", "restricted" in filename
    - IF suggests sensitive content:
        CLASSIFICATION = CONFIDENTIAL (conservative approach)
        CONFIDENCE = 40%
        ACTION = FLAG_FOR_ADMIN_REVIEW

IF file is encrypted (.gpg, .aes):
    - Assume RESTRICTED (defensive classification)
    - CONFIDENCE = 35%
    - ACTION = FLAG_FOR_ADMIN_REVIEW
```

### Edge Case 3: Code/Configuration Files
```
IF file extension in [.py, .js, .java, .cpp, .env, .conf]:
    SCAN FOR: API keys, database passwords, secrets
    
    IF pattern_match(API_KEY_PATTERNS):
        CLASSIFICATION = RESTRICTED
        RISK = CRITICAL
        ACTION = BLOCK
```

### Edge Case 4: False Positive Reduction
```
IF keyword detected but context suggests generic/example usage:
    - Reduce confidence by 30-50%
    - May reclassify to CONFIDENTIAL or INTERNAL
    
EXAMPLE: "How to create a strong password"
    - Contains "password" keyword
    - Context is educational
    - Confidence reduced: -50%
```

---

## 📋 SUMMARY TABLE

| Classification | Risk Level | Risk Score | When to Use | Action |
|---|---|---|---|---|
| **PUBLIC** | LOW | 0-5% | Empty/whitespace files, marketing, public docs | ALLOW (no restrictions) |
| **INTERNAL** | LOW-MEDIUM | 6-25% | Internal-only communications, system logs | ALLOW with LOGGING |
| **CONFIDENTIAL** | HIGH | 26-70% | Customer data, financial reports, contracts | WARN USER + LOG |
| **RESTRICTED** | CRITICAL | 71-100% | Trade secrets, credentials, regulated PII | BLOCK by default |

---

**Version:** 1.0  
**Last Updated:** January 11, 2026  
**Next Review:** April 11, 2026 (Quarterly)  
**Maintained By:** PRITRAK Security Team
