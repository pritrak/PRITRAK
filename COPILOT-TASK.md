# 🤖 GitHub Copilot Task: Real-Time File Classification

**Status:** Ready for Assignment  
**Priority:** CRITICAL  
**Estimated Time:** 4-6 hours  
**Outcome:** Production-ready C++ DLP classification system  

---

## Task Description

Implement a **real-time file classification engine** for PRITRAK Data Loss Prevention system. Users create/modify files on Windows disk, and your system must classify them instantly (<50ms) as PUBLIC/INTERNAL/CONFIDENTIAL/RESTRICTED based on AI-powered content analysis.

### Current Problem

```
User creates: C:\Users\PC\Desktop\Anti-Phishing\testt.txt
Logged to:    http://localhost:5173/dashboard/event-logs
Shows:        file_created, path, size
Missing:      Classification (currently EMPTY)
Required:     Classify in <50ms, invisibly
```

### Expected Result

```
User creates: testt.txt (empty)
T+50ms:       Dashboard shows: "PUBLIC (0/100) NONE"

User modifies with SSN: "123-45-6789"
T+50ms:       Dashboard shows: "CONFIDENTIAL (70/100) MEDIUM-HIGH"

User adds payroll data (100 rows of SSNs)
T+50ms:       Dashboard shows: "RESTRICTED (95/100) CRITICAL"
```

---

## Complete Specification

**READ THIS FIRST:**
- Main Spec: [AI-IMPLEMENTATION-PROMPT.md](./AI-IMPLEMENTATION-PROMPT.md) (21KB, 400+ lines)
- Quick Ref: [QUICK-REFERENCE-V2.md](./QUICK-REFERENCE-V2.md) (Scoring rules, patterns)
- Code Template: [IMPLEMENTATION-GUIDE-V2.md](./IMPLEMENTATION-GUIDE-V2.md) (C++ code)

---

## Your Tasks (In Order)

### Task 1: Phase 0 - Fast Filter
**Time: 30 minutes | Complexity: LOW**

**What it does:** Instantly classify 90% of files by extension/size

**Implement:**
```cpp
float ClassificationEngine::phase0_fast_filter(const std::string& file_path, size_t file_size)
```

**Rules to code:**
- If file is empty (0 bytes) → return 0 (PUBLIC)
- If extension is `.key`, `.pem`, `.env` → return 100 (RESTRICTED)
- If extension is `.txt` (not empty) → continue to next phase (return -1)
- If extension is `.sql` and file > 100MB → return 100 (RESTRICTED)
- If filename contains "password", "secret", "api_key" → return 95 (RESTRICTED)
- Otherwise → return -1 (no fast decision, continue)

**Test Cases:**
```
[] Empty testt.txt → 0 (PUBLIC) in <2ms
[] private_key.pem → 100 (RESTRICTED) in <1ms
[] config.env → 100 (RESTRICTED) in <1ms
[] budget.txt (not empty) → -1 (continue) in <2ms
```

**Success Criteria:**
- All tests pass
- Latency < 2ms for all cases
- No memory leaks

---

### Task 2: Validators
**Time: 45 minutes | Complexity: MEDIUM**

**What it does:** Validate that detected patterns are real (not false positives)

**Implement:**
```cpp
bool validate_luhn(const std::string& number);         // Credit cards
bool validate_ssn(const std::string& ssn);             // Social security
bool validate_french_nir(const std::string& nir);      // French ID
bool validate_iban(const std::string& iban);           // Bank accounts
```

**Luhn Algorithm (Credit Card):**
- Double every 2nd digit from right
- If doubled > 9, subtract 9
- Sum all digits
- If (sum % 10 == 0) → VALID

**SSN Validation:**
- Format: XXX-XX-XXXX
- NOT valid: 000-xx-xxxx, 666-xx-xxxx, 9xx-xx-xxxx, xxx-00-xxxx, xxx-xx-0000

**French NIR:**
- 13 digits
- First digit: 1 (male) or 2 (female)
- Check last 2 digits using mod 97

**Test Cases:**
```
[] Valid credit card: 4111111111111111 → true
[] Invalid credit card: 4111111111111112 → false
[] Valid SSN: 123-45-6789 → true
[] Invalid SSN: 000-00-0000 → false
[] Valid French NIR: 1 75 056 001 123 456 78 → true
```

---

### Task 3: Pattern Matcher
**Time: 60 minutes | Complexity: MEDIUM**

**What it does:** Find sensitive patterns in file content using regex

**Implement:**
```cpp
std::vector<Match> find_credit_cards(const std::string& content);
std::vector<Match> find_ssns(const std::string& content);
std::vector<Match> find_nirs(const std::string& content);
std::vector<Match> find_emails(const std::string& content);
```

**Regex Patterns:**

```regex
# Credit Card (Visa/Mastercard/Amex)
\b(?:4[0-9]{12}|5[1-5][0-9]{14}|3[47][0-9]{13})\b

# US SSN
\b(?!000|666|9\d{2})\d{3}-(?!00)\d{2}-(?!0000)\d{4}\b

# French NIR
\b[12]\s?(?:0[1-9]|1[0-2])\s?(?:(?:19|20)\d{2}|\d{2})\s?(?:0[1-95]|[2-8]\d|9[0-5])\s?\d{3}\s?\d{3}(?:\s?\d{2})?\b

# Email
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}
```

**Return Format:**
```cpp
struct Match {
    std::string value;     // "4111111111111111"
    size_t position;       // Character offset in file
};
```

**Test Cases:**
```
[] Find 2 credit cards in payroll CSV → 2 matches
[] Find 3 SSNs in employee list → 3 matches
[] No matches in empty file → 0 matches
[] Find all emails in contact list → correct count
```

---

### Task 4: Phase 1 - Pre-Analysis
**Time: 30 minutes | Complexity: LOW**

**What it does:** Score filename and directory for clues

**Implement:**
```cpp
float ClassificationEngine::phase1_pre_analysis(const std::string& file_path)
```

**Rules:**
```
Filename contains:
  "payroll" → +12
  "salary" → +12
  "password" → +15
  "api_key" → +20
  "ssn" → +25

Directory contains:
  "\\hr" → +10
  "\\finance" → +12
  "\\legal" → +10
  "\\executive" → +15
```

**Test Cases:**
```
[] payroll_2026.csv in \\finance → 12 + 12 = 24 points
[] budget.xlsx in C:\\Users\\Desktop → 0 points
[] password_reset.txt → 15 points
```

---

### Task 5: Phase 2 - Content Patterns
**Time: 90 minutes | Complexity: HIGH**

**What it does:** Scan file content for sensitive patterns

**Implement:**
```cpp
float ClassificationEngine::phase2_content_patterns(const std::string& content)
```

**Algorithm:**
1. Find all credit cards using regex
2. For each match, validate with Luhn (must pass)
3. Check negative context (not "example", "test", "sample")
4. If valid → add 20 points per credit card
5. Repeat for SSN (+25), French NIR (+35), keywords (+15 for "password", etc.)

**Scoring:**
```
Event                       | Points
========================================
Credit Card (Luhn valid)    | +20
SSN (format valid)          | +25
French NIR (mod97 valid)    | +35
Keyword "password"          | +15
Keyword "api_key"           | +20
Keyword "secret"            | +15
Keyword "salary"            | +8
```

**Negative Context Filter:**
If pattern is near "example", "test", "sample", "fake" → reduce by 50%

**Test Cases:**
```
[] File with valid SSN → +25 points
[] File with fake test CC + "example" context → ~0 points
[] Payroll CSV with 10 SSNs → 10 * 25 = 250 (diminished/capped)
[] Empty file → 0 points
```

---

### Task 6: Phase 3 - Context Logic
**Time: 30 minutes | Complexity: MEDIUM**

**What it does:** Apply business logic heuristics

**Implement:**
```cpp
float ClassificationEngine::phase3_context_logic(const std::string& file_path, const std::string& content)
```

**Rules:**
```
If file is .csv:
  - Count rows (newline characters)
  - If rows > 100 → +30 (bulk data export)
  - If rows > 1000 → +40

If file is .xlsx:
  - If contains data → +5

Language detection:
  - If majority French words → enable French patterns
```

**Test Cases:**
```
[] CSV with 2 rows → 0 adjustment
[] CSV with 100+ rows → +30 adjustment
[] CSV with 1000+ rows → +40 adjustment
```

---

### Task 7: Phase 4 - Final Decision
**Time: 30 minutes | Complexity: LOW**

**What it does:** Convert score (0-100) to classification

**Implement:**
```cpp
ClassificationResult ClassificationEngine::phase4_decision(float final_score)
```

**Classification Mapping:**
```
0-19:    PUBLIC         (confidence 100%, risk NONE)
20-49:   INTERNAL       (confidence 75%, risk LOW)
50-89:   CONFIDENTIAL   (confidence 85%, risk MEDIUM_HIGH)
90-100:  RESTRICTED     (confidence 95%, risk CRITICAL)
```

**Return Structure:**
```cpp
struct ClassificationResult {
    std::string classification;  // "PUBLIC", "INTERNAL", etc.
    float score;                 // 0-100, clamped
    float confidence;            // 0-100
    std::string explanation;     // "Why this classification"
    RiskLevel risk_level;        // NONE, LOW, MEDIUM_HIGH, CRITICAL
    int elapsed_ms;              // How long it took
};
```

---

### Task 8: Windows USN Journal Monitoring
**Time: 60 minutes | Complexity: HARD**

**What it does:** Detect file creation/modification events in real-time

**Implement:**
```cpp
class USNJournalMonitor {
public:
    bool start(std::function<void(const std::string&)> on_file_event);
    void stop();
    bool is_running() const;
private:
    void monitor_thread_func();
    bool is_system_file(const std::string& path) const;
};
```

**Algorithm:**
1. Open volume handle to C:\ 
2. Poll USN Journal every 100ms
3. Read USN records
4. Filter out system files (Windows/, AppData, Temp, .git, etc.)
5. For each user file, call on_file_event(file_path)

**Exclude Patterns:**
```
Windows, System32, AppData, LocalLow, Temp, $RECYCLE.BIN
Node_modules, .git, .vs, bin, obj, dist
ThumbDB.db, desktop.ini
```

---

### Task 9: WebSocket Client
**Time: 45 minutes | Complexity: MEDIUM**

**What it does:** Push classification results to React dashboard in real-time

**Implement:**
```cpp
class WebSocketClient {
public:
    bool connect(const std::string& uri = "ws://localhost:8080/ws/classification");
    bool is_connected() const;
    void disconnect();
    void push_classification(const std::string& file_path, 
                            const ClassificationResult& result);
};
```

**Message Format (JSON):**
```json
{
  "type": "classification_update",
  "file_path": "C:\\Users\\PC\\Desktop\\testt.txt",
  "classification": "PUBLIC",
  "score": 0,
  "confidence": 100,
  "explanation": "Empty file, no patterns detected",
  "risk_level": "NONE",
  "elapsed_ms": 2,
  "timestamp": "2026-01-11T13:36:45.000Z"
}
```

**Features:**
- Auto-reconnect on disconnect
- Timeout handling (60s)
- Graceful degradation if server unavailable

---

### Task 10: Main DLP Agent Service
**Time: 60 minutes | Complexity: MEDIUM**

**What it does:** Orchestrate all components as Windows service

**Implement:**
```cpp
class DLPAgent {
public:
    void start();
    void stop();
    bool is_running() const;
private:
    void process_file(const std::string& file_path);
};
```

**Workflow:**
1. USN Monitor detects file event
2. Queue file for classification
3. Processing thread:
   - Read file from disk
   - Run classification (Phase 0-4)
   - Push via WebSocket
   - Insert into SQLite database
   - Log to file
4. Repeat

**Database Schema:**
```sql
CREATE TABLE event_logs (
    id INTEGER PRIMARY KEY,
    file_path TEXT UNIQUE NOT NULL,
    classification TEXT NOT NULL,
    score REAL NOT NULL,
    confidence REAL NOT NULL,
    risk_level TEXT NOT NULL,
    explanation TEXT,
    elapsed_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

### Task 11: Unit Tests
**Time: 120 minutes | Complexity: MEDIUM**

**Implement tests for:**
```
[] Phase 0: Fast Filter
  [] Empty file → PUBLIC
  [] .key file → RESTRICTED
  [] Regular file → continue

[] Phase 1: Pre-Analysis
  [] Filename scoring works
  [] Directory scoring works
  [] No false positives

[] Phase 2: Content Patterns
  [] Credit card detection (with Luhn)
  [] SSN detection
  [] French NIR detection
  [] Negative context filtering

[] Validators
  [] Luhn algorithm
  [] SSN validation
  [] French NIR validation

[] Pattern Matcher
  [] Regex patterns match correctly
  [] No spurious matches

[] Full Classification
  [] E2E: Empty file → PUBLIC (0/100) in <2ms
  [] E2E: Payroll CSV → RESTRICTED (95/100) in <50ms
  [] E2E: File modification → reclassification
```

**Coverage Target:** >85%

---

### Task 12: Compilation & Build
**Time: 30 minutes | Complexity: LOW**

**What it does:** Compile all code into pritrak_dlp.exe

**Instructions:**
1. Install vcpkg dependencies
2. Create CMakeLists.txt
3. Compile: `cmake --build . --config Release`
4. Run: `ctest --output-on-failure`
5. Result: C++17 compliant, no warnings, all tests pass

---

## Completion Checklist

Do NOT submit until ALL are checked:

- [ ] Phase 0: Fast Filter implemented & tested
- [ ] Phase 1: Pre-Analysis implemented & tested
- [ ] Phase 2: Content Patterns implemented & tested
- [ ] Phase 3: Context Logic implemented & tested
- [ ] Phase 4: Final Decision implemented & tested
- [ ] Validators (Luhn, SSN, NIR, IBAN) implemented & tested
- [ ] Pattern Matcher (regex + Match struct) implemented & tested
- [ ] USN Journal Monitor implemented & tested
- [ ] WebSocket Client implemented & tested
- [ ] DLP Agent Service (main orchestrator) implemented & tested
- [ ] Unit tests created with >85% coverage
- [ ] All tests pass
- [ ] Code compiles without warnings
- [ ] Performance benchmarks met:
  - [ ] Phase 0: <2ms
  - [ ] Phase 1: <5ms
  - [ ] Phase 2: <30ms
  - [ ] Total p95: <50ms
- [ ] False positive rate <1.5%
- [ ] No memory leaks
- [ ] Windows service installs successfully
- [ ] Integration tests pass (end-to-end)
- [ ] Dashboard receives classifications in real-time
- [ ] Database stores all classifications
- [ ] File modification triggers reclassification
- [ ] No TODOs in code
- [ ] Code is production-ready

---

## Resources

**Complete Specifications:**
- [AI-IMPLEMENTATION-PROMPT.md](./AI-IMPLEMENTATION-PROMPT.md) ← **START HERE**
- [QUICK-REFERENCE-V2.md](./QUICK-REFERENCE-V2.md) ← Scoring rules
- [IMPLEMENTATION-GUIDE-V2.md](./IMPLEMENTATION-GUIDE-V2.md) ← C++ code templates
- [DEPLOYMENT-CHECKLIST-V2.md](./DEPLOYMENT-CHECKLIST-V2.md) ← Deployment guide

**References:**
- C++ regex: https://cppreference.com/w/cpp/regex
- Luhn algorithm: https://en.wikipedia.org/wiki/Luhn_algorithm
- USN Journal: https://docs.microsoft.com/en-us/windows/win32/fileio/change-journals
- WebSocket: https://github.com/zaphoyd/websocketpp

---

## Notes for AI Agent

⚠️ **CRITICAL:** This is a PRODUCTION SYSTEM, not a prototype.

- **Do NOT stop coding until 100% complete**
- **Every feature must be production-ready**
- **No TODOs or placeholders**
- **No false positives (use negative context filtering)**
- **Every millisecond counts (performance matters)**
- **Test thoroughly (unit + integration + stress)**
- **Write clean, maintainable code**
- **Follow C++17 best practices**
- **Handle errors gracefully**
- **Document the code**

**Expected Output:**
1. Fully compiled C++ executable (pritrak_dlp.exe)
2. All unit tests passing
3. Windows service installable
4. Real-time classification working
5. Dashboard populated with classifications
6. Performance targets met
7. Zero false positives
8. Production deployment ready

**Status:** 🚨 DO NOT COMMIT INCOMPLETE CODE

---

## Success Criteria

**You are done when:**

1. Empty file classified as PUBLIC in <2ms ✅
2. Payroll CSV classified as RESTRICTED in <50ms ✅
3. File modification triggers reclassification ✅
4. Dashboard shows classifications in real-time ✅
5. All unit tests pass with >85% coverage ✅
6. No warnings on compilation ✅
7. Windows service runs successfully ✅
8. Performance benchmarks met ✅
9. False positive rate <1.5% ✅
10. Ready for production deployment ✅

---

**Assigned to:** GitHub Copilot  
**Priority:** CRITICAL  
**Deadline:** January 12, 2026  
**Status:** READY TO START  

**BEGIN NOW. DO NOT STOP UNTIL 100% COMPLETE.** 🚀
