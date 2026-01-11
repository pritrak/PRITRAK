# QUICK REFERENCE V2.0 - Classification System

## Classification Levels (Simple)

```
SCORE RANGE    |  CLASSIFICATION  |  RISK LEVEL      |  ACTION
========================================================================
0 - 19         |  PUBLIC          |  NONE            |  Allow all access
20 - 49        |  INTERNAL        |  LOW             |  Log access
50 - 89        |  CONFIDENTIAL    |  MEDIUM-HIGH     |  Restrict to dept
90 - 100       |  RESTRICTED      |  CRITICAL        |  Block/Alert
```

---

## Scoring Rules Quick Lookup

### Extension Rules (Phase 0)

```
EXTENSION      |  SCORE  |  DECISION        |  LATENCY
=============================================================
.key, .pem     |  100    |  Instant BLOCK   |  <1ms
.p12, .pfx     |  100    |  Instant BLOCK   |  <1ms
.gpg, .aes     |  100    |  Instant BLOCK   |  <1ms
.env           |  100    |  Instant BLOCK   |  <1ms
.sql           |  95     |  Instant BLOCK   |  <1ms
.sql.bak       |  95     |  Instant BLOCK   |  <1ms
.xlsx, .csv    |  +5     |  Continue        |  Continue to Phase 1
.txt           |  0      |  Instant ALLOW   |  <1ms (if empty)
.log           |  +3     |  Continue        |  Continue
.tmp, .cache   |  0      |  Instant ALLOW   |  <1ms
```

### Filename Keywords (Phase 1)

```
KEYWORD              |  SCORE ADD  |  EXAMPLES
=====================================================================
password             |  +15        |  password.txt, pwd_2026.xlsx
secret               |  +15        |  client_secrets.json
api_key              |  +20        |  api_key_prod.env
private_key          |  +20        |  id_rsa
payroll              |  +12        |  payroll_2026_q1.csv
salary               |  +12        |  salary_review.xlsx
customer             |  +8         |  customers_list.csv
invoice              |  +8         |  invoice_details.xlsx
financial            |  +10        |  financial_report.pdf
pii                  |  +18        |  pii_data.csv
ssn                  |  +25        |  ssn_list.xlsx
credit_card          |  +20        |  cc_numbers.txt
```

### Directory Context (Phase 1)

```
DIRECTORY            |  SCORE ADD  |  RATIONALE
=====================================================================
\\hr                  |  +10        |  HR department files
\\finance             |  +12        |  Financial data
\\legal               |  +10        |  Legal documents
\\executive           |  +15        |  C-level sensitive
\\secret              |  +20        |  Explicit
\\classified          |  +20        |  Explicit
```

### Content Patterns (Phase 2)

#### Regex Patterns

```regex
# US Credit Card (Visa, Mastercard, Amex)
(?:4[0-9]{12}|5[1-5][0-9]{14}|3[47][0-9]{13})
→ Score: +20 (if Luhn valid)

# US Social Security Number (XXX-XX-XXXX)
(?!000|666|9\d{2})\d{3}-(?!00)\d{2}-(?!0000)\d{4}
→ Score: +25

# Email Address
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}
→ Score: +3 (per email)

# Phone Number (US +1-XXX-XXX-XXXX)
\+?1?[-.]?\(?[0-9]{3}\)?[-.]?[0-9]{3}[-.]?[0-9]{4}
→ Score: +2 (per phone)
```

#### Validators

```
TYPE             |  VALIDATION METHOD
======================================================================
Credit Card      |  Luhn Algorithm (mod 10 checksum)
SSN              |  Not 000-00-0000, 666-xx-xxxx, 9xx-xx-xxxx
French NIR       |  13 digits with checksum: 97 - (concatenated % 97)
IBAN             |  Mod-97 algorithm + valid country code
Email            |  RFC 5322 (simplified)
Phone            |  Length check (10-15 digits)
```

#### Pattern Weights

```
PATTERN                 |  WEIGHT  |  NOTES
==================================================================
Credit Card (Luhn OK)   |  +20     |  Only if Luhn passes
SSN                     |  +25     |  High confidence
French NIR              |  +35     |  Very high confidence
IBAN                    |  +18     |  Bank account
Keywords:
  - password            |  +15
  - api_key             |  +20
  - secret              |  +15
  - salary              |  +8
  - employee            |  +5
```

---

## Negative Context Filtering

**IMPORTANT:** If pattern matches but context contains these keywords, reduce score by 50%

```
KEYWORD              |  REASON
==================================================================
example              |  Code example
test                 |  Test data
sample               |  Sample data
fake                 |  Fake/dummy
mock                 |  Mock for testing
random               |  Placeholder
dummy                |  Dummy value
placeholder          |  Placeholder
template             |  Template
```

**Example:**

```
File: test_data.csv
Content contains: 123-45-6789 (SSN pattern) in section "# FAKE TEST DATA"

Phase 2 calculation:
  SSN match:        +25
  Negative context: -12 (50% reduction for "fake" + "test")
  Final:            +13
```

---

## French Data Patterns

### NIR (Numéro d'Inscription Répertoire / Secu)

**Format:** 1 (or 2) + 2 (month) + 2 or 4 (year) + 2 or 3 (dept) + 3 (commune) + 3 (order) + 2 (key)

**Example:** 1 75 056 001 123 456 78

**Regex:**
```regex
\b[12]\s?(?:0[1-9]|1[0-2])\s?(?:(?:19|20)\d{2}|\d{2})\s?(?:0[1-95]|[2-8]\d|9[0-5])\s?\d{3}\s?\d{3}(?:\s?\d{2})?\b
```

**Validation:**
```
1. Extract 13 digits
2. Calculate: key = 97 - (number % 97)
3. Last 2 digits must match key
```

**Score:** +35 (if valid)

### French IBAN

**Format:** FR + 2 check digits + 5 bank code + 5 branch code + 11 account + 2 key

**Example:** FR1420041010050500013M02606

**Regex:**
```regex
FR[0-9]{2}[A-Z0-9]{23}
```

**Score:** +18 (if valid IBAN)

---

## Implementation Checklist

### Phase 0 (< 2ms)
- [ ] Extension whitelist/blacklist
- [ ] File size checks
- [ ] Sensitive filenames detection

### Phase 1 (< 5ms)
- [ ] Filename keyword scoring
- [ ] Directory context scoring
- [ ] File type heuristics

### Phase 2 (< 30ms)
- [ ] Regex pattern matching (credit cards, SSN, NIR, IBAN)
- [ ] Luhn algorithm validation
- [ ] French NIR validation
- [ ] Negative context filtering

### Phase 2.5 (optional, < 150ms)
- [ ] GLiNER AI model call (only if 50 ≤ score < 90)
- [ ] Timeout handling (conservative bias)

### Phase 3 (< 10ms)
- [ ] File type heuristics
- [ ] Proximity boosting

### Phase 4 (< 3ms)
- [ ] Score clamping [0, 100]
- [ ] Classification mapping

---

## Real-Time Dashboard Integration

### WebSocket Message Format

```json
{
  "type": "classification_update",
  "file_path": "C:\\Users\\PC\\Desktop\\Anti-Phishing\\testt.txt",
  "classification": "PUBLIC",
  "score": 0,
  "confidence": 100,
  "explanation": "Empty file, no patterns detected",
  "risk_level": "NONE",
  "elapsed_ms": 2,
  "timestamp": "2026-01-11T13:36:45.000Z"
}
```

### Database Schema

```sql
INSERT OR REPLACE INTO event_logs 
(file_path, classification, score, confidence, explanation, risk_level, elapsed_ms, updated_at)
VALUES (?, ?, ?, ?, ?, ?, ?, CURRENT_TIMESTAMP);
```

---

## Performance Targets

```
Operations          |  Target  |  Measurement
==================================================================
Phase 0             |  < 2ms   |  90% of files
Phase 1             |  < 5ms   |  Per file
Phase 2             |  < 30ms  |  Per file
Phase 2.5 (AI)      |  < 150ms |  10% of ambiguous files
Phase 3             |  < 10ms  |  Per file
Phase 4             |  < 3ms   |  Per file
---
Total p95           |  < 50ms  |  User won't notice
WebSocket push      |  < 10ms  |  Real-time feel
Database insert     |  < 5ms   |  SQLite write
Dashboard render    |  < 10ms  |  React update
---
End-to-end          |  < 100ms |  File creation to display
```

---

## Scoring Formula

```
Final Score = Phase0_FastFilter()
            + Phase1_PreAnalysis()
            + Phase2_ContentPatterns()
            + Phase25_AISidecar()  [optional]
            + Phase3_ContextLogic()
            - NegativeContextReduction()

Clamped to [0, 100]
```

---

## Examples

### Example 1: Empty File (testt.txt)

```
T+0ms:   User creates testt.txt (0 bytes)
T+2ms:   Phase 0: fileSize == 0
         → INSTANT ALLOW
         → Score: 0
         → Classification: PUBLIC
         → Confidence: 100%

Result: PUBLIC (0/100) NONE ✅
```

### Example 2: Private Key

```
T+0ms:   User creates private_key.pem
T+1ms:   Phase 0: extension == .pem
         → INSTANT BLOCK
         → Score: 100
         → Classification: RESTRICTED

Result: RESTRICTED (100/100) CRITICAL ✅
```

### Example 3: Payroll CSV

```
T+0ms:   User creates payroll_2026.csv
         Content: Employee,SSN,Salary
                 Alice,123-45-6789,85000

T+2ms:   Phase 0: .csv file
         → No fast decision (continue)
         → Score: 0

T+7ms:   Phase 1: filename "payroll"
         → Score += 12
         → Score: 12

T+45ms:  Phase 2: Content patterns
         - SSN pattern found: 123-45-6789
         - Luhn validation: PASS
         → Score += 25
         - Keyword "Salary"
         → Score += 8
         → Score: 45
         - Negative context: NONE

T+50ms:  Phase 3: Context
         - CSV file with 2 rows
         → Score += 5 (small dataset)
         → Score: 50

T+53ms:  Phase 4: Final Decision
         - Score: 50 >= 50
         → Classification: CONFIDENTIAL
         → Risk Level: MEDIUM-HIGH

Result: CONFIDENTIAL (50/100) MEDIUM-HIGH ✅
```

### Example 4: Modified Payroll (50→90)

```
T+0ms:   User modifies payroll_2026.csv
         Adds 100 more employees with full SSNs

T+50ms:  Phase 2: Content patterns
         - 100x SSN patterns found
         - All pass Luhn
         → Score += (100 * 25) BUT... clamped at different rates
         - Keyword "Salary" appears 100x
         → Score += (100 * 1) [diminishing return]

T+60ms:  Phase 3: Context
         - CSV file with 100 rows (bulk export)
         → Score += 30

T+65ms:  Phase 4: Final Decision
         - Score: 90+ ≥ 90
         → Classification: RESTRICTED
         → Risk Level: CRITICAL

Result: RESTRICTED (95/100) CRITICAL ✅
```

---

## Debugging Tips

**File not classified?**
1. Check if file path matches Windows filter (not Temp, not AppData)
2. Check if USN Journal is enabled on volume
3. Check if C++ service is running

**Wrong classification?**
1. Run file through Phase 0-4 manually
2. Check regex patterns (use online regex tester)
3. Check negative context filtering
4. Check Phase 2.5 AI sidecar if score 50-90

**Slow classification?**
1. Check Phase 2 latency (usually <30ms)
2. Check Phase 2.5 AI timeout (set to 100ms max)
3. Check file read latency (large files)

---

## References

- **Full Implementation:** AI-IMPLEMENTATION-PROMPT.md
- **C++ Code Template:** IMPLEMENTATION-GUIDE-V2.md
- **Deployment:** DEPLOYMENT-CHECKLIST-V2.md

---

*Last Updated: January 11, 2026*
