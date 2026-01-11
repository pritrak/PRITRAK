# 🤖 PRITRAK V2.0 Real-Time Classification System - AI Implementation Prompt

**CRITICAL: This prompt should be fed to GitHub Copilot or Claude as a single continuous task. The AI should NOT STOP until 100% completion.**

---

## EXECUTIVE BRIEF

You are implementing a **real-time file classification engine** for PRITRAK Data Loss Prevention (DLP) system. Users create/modify files on Windows (C:\Users\...), and your system must classify them instantly (within 50ms latency) as PUBLIC/INTERNAL/CONFIDENTIAL/RESTRICTED.

**Current Problem:**
- File created: `testt.txt`
- Logged to: `http://localhost:5173/dashboard/event-logs`
- Classification field: **EMPTY** (not being populated)
- Required: Classification happens in real-time, invisibly to user, with score 0-100

**Your Mission:**
1. Implement real-time file classification in C++ DLP agent
2. Hook into file system events (Windows IRP monitoring)
3. Classify before file is fully created (predictive)
4. Update dashboard via WebSocket in <50ms
5. Handle empty→modified transitions seamlessly
6. Deploy as production-ready microservice

---

## PART 1: SYSTEM ARCHITECTURE

### Data Flow (What Must Happen)

```
User Action: File Created/Modified on disk
    ↓
    ↓ [Windows IRP/USN Journal captures event]
    ↓
C++ DLP Agent receives event
    ↓
    ├─ Phase 0: Fast Filter (< 2ms)
    │  - Check extension, size
    │  - If empty .txt → score = 0 (PUBLIC)
    │  - If .key file → score = 100 (RESTRICTED)
    │
    ├─ Phase 1: Pre-Analysis (< 5ms)
    │  - Filename scoring
    │  - Directory context
    │  - File type detection
    │
    ├─ Phase 2: Content Patterns (< 30ms)
    │  - Regex scanning (secrets, PII)
    │  - Validator checks (Luhn, IBAN, NIR)
    │  - Negative context filtering
    │
    ├─ Phase 2.5: AI Sidecar (IF 50≤score<90, < 150ms)
    │  - Call GLiNER for disambiguation
    │
    ├─ Phase 3: Context Logic (< 10ms)
    │  - File type heuristics
    │  - Proximity boosting
    │
    └─ Phase 4: Final Decision (< 3ms)
       - Clamp score to [0, 100]
       - Map to classification
    ↓
Result: {classification, score, confidence}
    ↓
    ├─ [WebSocket push to dashboard]
    ├─ [Database store]
    └─ [Audit log]
    ↓
Dashboard shows classification within 50ms
```

### Integration Points

**RECOMMENDED: Windows USN Journal + WebSocket**

```cpp
// File events flow:
User creates testt.txt
  ↓
Windows USN Journal detects IRP_MJ_CREATE
  ↓
C++ DLP Agent (Service) polls USN Journal every 100ms
  ↓
Matches file pattern (not system file, not temp)
  ↓
Reads file (if accessible, or predicts from path)
  ↓
Classification Engine (Phases 0-4) scores file
  ↓
WebSocket client sends to backend
  ↓
Backend stores in SQLite
  ↓
Dashboard receives update, re-renders table
  ↓
User sees: "PUBLIC (0/100)" or "RESTRICTED (100/100)"
```

---

## PART 2: IMPLEMENTATION SPEC

### 2.1 Phase 0: Fast Filter (Must Be < 2ms)

```cpp
// Eliminate 90% of files instantly

struct FastFilterResult {
    bool decided;
    float score;
    std::string classification;
};

FastFilterResult Phase0_FastFilter(const std::string& filePath, size_t fileSize) {
    // RULE 1: Instant BLOCK extensions
    const std::vector<std::string> INSTANT_BLOCK = {
        ".key", ".pem", ".p12", ".pfx", ".ppk",
        ".sql", ".sql.bak", ".env", ".env.production",
        ".gpg", ".aes"
    };
    
    // RULE 2: Instant ALLOW extensions  
    const std::vector<std::string> INSTANT_ALLOW = {
        ".txt", ".log", ".tmp", ".cache"
    };
    
    // RULE 3: Empty file
    if (fileSize == 0) {
        return {true, 0.0f, "PUBLIC"};
    }
    
    // RULE 4: Large SQL dump
    if (fileSize > 100_MB && (ends_with(filePath, ".sql") || ends_with(filePath, ".db"))) {
        return {true, 100.0f, "RESTRICTED"};
    }
    
    // RULE 5: Check extensions
    std::string ext = get_extension(filePath);
    
    if (contains(INSTANT_BLOCK, ext)) {
        return {true, 100.0f, "RESTRICTED"};
    }
    if (contains(INSTANT_ALLOW, ext)) {
        return {true, 0.0f, "PUBLIC"};
    }
    
    // RULE 6: Sensitive filenames
    std::string filename = to_lower(get_filename(filePath));
    if (filename.find("password") != std::string::npos ||
        filename.find("secret") != std::string::npos ||
        filename.find("api_key") != std::string::npos) {
        return {true, 95.0f, "RESTRICTED"};
    }
    
    // No fast decision
    return {false, 0.0f, ""};
}
```

**Expected:** Empty testt.txt → <2ms → "PUBLIC"

### 2.2 Phase 1: Pre-Analysis (< 5ms)

```cpp
float Phase1_PreAnalysis(const std::string& filePath) {
    float score = 0.0f;
    
    // Filename scoring
    std::string filename = to_lower(get_filename(filePath));
    if (filename.find("payroll") != std::string::npos) score += 12;
    if (filename.find("salary") != std::string::npos) score += 12;
    if (filename.find("customer") != std::string::npos) score += 8;
    if (filename.find("invoice") != std::string::npos) score += 8;
    
    // Directory scoring
    std::string dir = to_lower(filePath);
    if (dir.find("\\hr") != std::string::npos) score += 10;
    if (dir.find("\\finance") != std::string::npos) score += 12;
    if (dir.find("\\legal") != std::string::npos) score += 10;
    if (dir.find("\\executive") != std::string::npos) score += 15;
    
    // File extension scoring
    if (ends_with(filePath, ".xlsx") || ends_with(filePath, ".csv")) {
        score += 5;  // Often contains PII
    }
    
    return score;
}
```

### 2.3 Phase 2: Content Patterns (< 30ms)

```cpp
struct PatternMatch {
    std::string type;    // "CREDIT_CARD", "SSN", etc.
    float weight;        // 0-100
    bool valid;          // Passed validator
};

float Phase2_ContentPatterns(const std::string& content) {
    float score = 0.0f;
    
    // Pattern 1: Credit cards (with Luhn validation)
    std::regex creditCardRegex(R"(\b(?:4[0-9]{12}|5[1-5][0-9]{14}|3[47][0-9]{13})\b)");
    std::sregex_iterator it(content.begin(), content.end(), creditCardRegex);
    std::sregex_iterator end;
    
    while (it != end) {
        std::string match = it->str();
        
        // Negative context check
        std::string context = get_context(content, it->position(), 200);
        if (context.find("example") == std::string::npos &&
            context.find("test") == std::string::npos &&
            context.find("sample") == std::string::npos) {
            
            // Validate with Luhn
            if (validate_luhn(match)) {
                score += 20;  // Weighted score
            }
        }
        ++it;
    }
    
    // Pattern 2: US SSN
    std::regex ssnRegex(R"(\b(?!000|666|9\d{2})\d{3}-(?!00)\d{2}-(?!0000)\d{4}\b)");
    it = std::sregex_iterator(content.begin(), content.end(), ssnRegex);
    
    while (it != end) {
        score += 25;  // SSN = high risk
        ++it;
    }
    
    // Pattern 3: French NIR (Sécu)
    std::regex nirRegex(R"(\b[12]\s?(?:0[1-9]|1[0-2])\s?(?:(?:19|20)\d{2}|\d{2})\s?(?:0[1-95]|[2-8]\d|9[0-5])\s?\d{3}\s?\d{3}(?:\s?\d{2})?\b)");
    it = std::sregex_iterator(content.begin(), content.end(), nirRegex);
    
    while (it != end) {
        std::string match = it->str();
        if (validate_french_nir(match)) {
            score += 35;  // French NIR = very high risk
        }
        ++it;
    }
    
    // Pattern 4: Keywords
    if (content.find("password") != std::string::npos) score += 15;
    if (content.find("api_key") != std::string::npos) score += 20;
    if (content.find("secret") != std::string::npos) score += 15;
    if (content.find("salary") != std::string::npos) score += 8;
    if (content.find("employee") != std::string::npos) score += 5;
    
    return score;
}
```

### 2.4 Phase 2.5: AI Sidecar (Optional, IF 50 ≤ score < 90)

```cpp
float Phase25_AISidecar(float currentScore, const std::string& snippet) {
    // Only call if score is ambiguous
    if (currentScore < 50 || currentScore >= 90) {
        return 0.0f;  // No adjustment
    }
    
    // Call AI service with timeout
    try {
        auto response = call_gliner_api(snippet, 
            {"RealCredential", "TestData", "CodeExample"});
        
        if (response["TestData"] > 0.8f) {
            return -30.0f;  // DOWNGRADE (test data)
        } else if (response["RealCredential"] > 0.85f) {
            return +20.0f;  // UPGRADE (real credential)
        }
    } catch (...) {
        // AI service down → conservative bias
        return 0.0f;  // No adjustment
    }
    
    return 0.0f;  // Uncertain → no change
}
```

### 2.5 Phase 3: Context Logic (< 10ms)

```cpp
float Phase3_ContextLogic(const std::string& filePath, const std::string& content) {
    float adjustment = 0.0f;
    
    // File type heuristics
    if (ends_with(filePath, ".csv") && count_rows(content) > 100) {
        adjustment += 30;  // Bulk data export = higher risk
    }
    
    // Language detection
    float frenchScore = count_french_words(content);
    float englishScore = count_english_words(content);
    
    if (frenchScore > englishScore) {
        // Apply French patterns (already done in Phase 2)
    }
    
    // Proximity boosting
    // If keyword near PII, boost score
    
    return adjustment;
}
```

### 2.6 Phase 4: Final Decision (< 3ms)

```cpp
struct ClassificationResult {
    std::string classification;  // "PUBLIC", "INTERNAL", "CONFIDENTIAL", "RESTRICTED"
    float score;                  // 0-100
    float confidence;             // 0-100
    std::string explanation;      // "Why this classification"
    std::string riskLevel;        // "NONE", "LOW", "MEDIUM-HIGH", "CRITICAL"
};

ClassificationResult Phase4_Decision(float finalScore) {
    // Clamp to [0, 100]
    finalScore = std::max(0.0f, std::min(100.0f, finalScore));
    
    if (finalScore >= 90) {
        return {
            "RESTRICTED",
            finalScore,
            95.0f,
            "File contains classified/sensitive data",
            "CRITICAL"
        };
    } else if (finalScore >= 50) {
        return {
            "CONFIDENTIAL",
            finalScore,
            85.0f,
            "File contains restricted information",
            "MEDIUM-HIGH"
        };
    } else if (finalScore >= 20) {
        return {
            "INTERNAL",
            finalScore,
            75.0f,
            "File for internal use only",
            "LOW"
        };
    } else {
        return {
            "PUBLIC",
            finalScore,
            100.0f,
            "No sensitive data detected",
            "NONE"
        };
    }
}
```

### 2.7 Windows USN Journal Monitor

```cpp
class USNJournalMonitor {
private:
    HANDLE hVolume;
    std::thread monitorThread;
    std::function<void(const std::string&)> onFileEvent;
    
public:
    void start(std::function<void(const std::string&)> callback) {
        onFileEvent = callback;
        
        // Open C:\ volume
        hVolume = CreateFileA("C:\\", GENERIC_READ, 
                             FILE_SHARE_READ | FILE_SHARE_WRITE,
                             NULL, OPEN_EXISTING, 0, NULL);
        
        // Monitor USN Journal in separate thread
        monitorThread = std::thread([this]() {
            monitor_usn_journal();
        });
    }
    
private:
    void monitor_usn_journal() {
        // Read USN records every 100ms
        // On file create/modify: call onFileEvent(filePath)
        // Skip system files (Windows, AppData, Temp, etc.)
    }
};
```

### 2.8 WebSocket Client (Real-Time Dashboard Push)

```cpp
class WebSocketClient {
private:
    WebSocket ws;
    bool isConnected = false;
    
public:
    void connect(const std::string& uri = "ws://localhost:8080/ws/classification") {
        // Connect to backend WebSocket
        ws.connect(uri);
        isConnected = true;
    }
    
    void pushClassification(const std::string& filePath, 
                           const ClassificationResult& result) {
        if (!isConnected) return;
        
        // Build JSON
        Json::Value payload;
        payload["type"] = "classification_update";
        payload["file_path"] = filePath;
        payload["classification"] = result.classification;
        payload["score"] = result.score;
        payload["confidence"] = result.confidence;
        payload["explanation"] = result.explanation;
        payload["risk_level"] = result.riskLevel;
        payload["timestamp"] = get_iso8601_timestamp();
        
        // Send
        ws.send(Json::FastWriter().write(payload));
    }
};
```

### 2.9 Main DLP Agent Service

```cpp
class DLPAgent {
private:
    USNJournalMonitor monitor;
    ClassificationEngine engine;
    WebSocketClient wsClient;
    DatabaseClient dbClient;  // SQLite
    std::queue<std::string> processQueue;
    std::thread processingThread;
    
public:
    void start() {
        // Connect to WebSocket
        wsClient.connect("ws://localhost:8080/ws/classification");
        
        // Connect to database
        dbClient.open("sqlite:///C:/ProgramData/PRITRAK/classifications.db");
        
        // Start monitoring
        monitor.start([this](const std::string& filePath) {
            processQueue.push(filePath);  // Queue file for classification
        });
        
        // Start processing thread
        processingThread = std::thread([this]() {
            while (true) {
                if (!processQueue.empty()) {
                    std::string filePath = processQueue.front();
                    processQueue.pop();
                    
                    try {
                        // Classify
                        auto result = engine.classify(filePath);
                        
                        // Push to dashboard
                        wsClient.pushClassification(filePath, result);
                        
                        // Store in database
                        dbClient.insert(filePath, result);
                        
                        // Alert if RESTRICTED
                        if (result.classification == "RESTRICTED") {
                            log_incident(filePath, result);
                        }
                    } catch (const std::exception& e) {
                        std::cerr << "Error: " << e.what() << std::endl;
                    }
                }
                std::this_thread::sleep_for(std::chrono::milliseconds(10));
            }
        });
    }
};
```

---

## PART 3: INTEGRATION WITH YOUR DASHBOARD

### Backend: Accept Classification Updates

```typescript
// backend/src/routes/classifications.ts

const wss = new WebSocketServer({port: 8080, path: '/ws/classification'});

wss.on('connection', (ws) => {
    ws.on('message', (data) => {
        const {type, file_path, classification, score, confidence, explanation, risk_level} = JSON.parse(data);
        
        if (type === 'classification_update') {
            // Store in SQLite
            db.run(
                `INSERT OR REPLACE INTO event_logs 
                 (file_path, classification, score, confidence, explanation, risk_level, updated_at)
                 VALUES (?, ?, ?, ?, ?, ?, CURRENT_TIMESTAMP)`,
                [file_path, classification, score, confidence, explanation, risk_level]
            );
            
            // Broadcast to dashboard clients
            broadcast_to_dashboard({type, file_path, classification, score, confidence, risk_level});
        }
    });
});
```

### Frontend: Display Classifications

```typescript
// frontend/src/components/EventLogs.tsx

export function EventLogs() {
    const [events, setEvents] = useState([]);
    
    useEffect(() => {
        // WebSocket real-time updates
        const ws = new WebSocket('ws://localhost:8080/ws/classification');
        
        ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            if (data.type === 'classification_update') {
                // Update table row
                setEvents(prev => {
                    const idx = prev.findIndex(e => e.file_path === data.file_path);
                    if (idx >= 0) {
                        prev[idx] = {file_path: data.file_path, ...data};
                    } else {
                        prev.unshift({file_path: data.file_path, ...data});
                    }
                    return [...prev];
                });
            }
        };
    }, []);
    
    return (
        <table>
            <tbody>
                {events.map(event => (
                    <tr key={event.file_path}>
                        <td>file_created</td>
                        <td>{event.file_path}</td>
                        <td>
                            <span style={{color: get_color(event.classification)}}>
                                {event.classification} ({event.score.toFixed(0)}/100)
                            </span>
                        </td>
                        <td>{event.risk_level}</td>
                    </tr>
                ))}
            </tbody>
        </table>
    );
}
```

---

## PART 4: DATABASE SCHEMA

```sql
CREATE TABLE event_logs (
    id INTEGER PRIMARY KEY,
    file_path TEXT UNIQUE NOT NULL,
    classification TEXT NOT NULL,  -- PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
    score REAL NOT NULL,           -- 0-100
    confidence REAL NOT NULL,      -- 0-100
    risk_level TEXT NOT NULL,      -- NONE, LOW, MEDIUM-HIGH, CRITICAL
    explanation TEXT,
    elapsed_ms INTEGER,            -- Classification latency
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## PART 5: TESTING CHECKLIST

- [ ] Empty .txt file → <2ms → PUBLIC (0/100)
- [ ] .key file → <2ms → RESTRICTED (100/100)
- [ ] Payroll CSV with SSNs → <50ms → RESTRICTED (>90/100)
- [ ] README.md with test credit card → <50ms → INTERNAL (<50/100)
- [ ] Empty .xlsx → PUBLIC (0/100)
- [ ] Modified .xlsx with data → CONFIDENTIAL or RESTRICTED
- [ ] WebSocket push within 50ms of classification
- [ ] Dashboard shows classification in real-time
- [ ] Database stores all classifications
- [ ] False positive rate <1.5% on test set
- [ ] Accuracy >95%
- [ ] Memory <200MB
- [ ] CPU <10% per file

---

## PART 6: EXAMPLE SCENARIOS

### Scenario 1: Empty File

```
T+0ms:  User creates testt.txt (0 bytes)
T+2ms:  USN Journal detects event
T+3ms:  Phase 0: size == 0 → ALLOW
T+4ms:  WebSocket push
T+10ms: Dashboard shows "PUBLIC (0/100)"
```

### Scenario 2: Payroll Data

```
T+0ms:  User pastes payroll CSV:
        Employee,SSN,Salary
        Alice,123-45-6789,85000
T+10ms: USN Journal detects write
T+12ms: Phase 0: .csv file → continue
T+15ms: Phase 1: filename scoring
T+45ms: Phase 2: 2x SSN matches (+50), keywords (+16) = 116
T+50ms: Phase 4: 116 >= 90 → RESTRICTED
T+51ms: WebSocket push
T+55ms: Dashboard shows "RESTRICTED (100/100) CRITICAL"
```

---

## PART 7: PERFORMANCE TARGETS

```
Phase 0:           < 2ms  (90% of files stop here)
Phase 1:           < 5ms
Phase 2:           < 30ms
Phase 2.5:         < 150ms (only 10% of files, ambiguous)
Phase 3:           < 10ms
Phase 4:           < 3ms
─────────────────────────
Total p95:         < 50ms  (user won't notice delay)
WebSocket push:    < 10ms
Database insert:   < 5ms
Dashboard render:  < 10ms
─────────────────────────
End-to-end:        < 100ms (file creation to dashboard display)
```

---

## PART 8: REFERENCES

**Complete specs:**
- CLASSIFICATION-MATRIX-V2.md (regex patterns, scoring rules)
- IMPLEMENTATION-GUIDE-V2.md (C++ code, deployment)
- QUICK-REFERENCE-V2.md (validator reference)

**Your dashboard:**
- Frontend: http://localhost:5173/dashboard/event-logs
- Backend: Run local Node.js server on port 8080
- Database: SQLite at C:/ProgramData/PRITRAK/classifications.db

---

## FINAL INSTRUCTIONS FOR AI AGENT

**You are not making suggestions. You are implementing a PRODUCTION SYSTEM.**

**Do NOT stop until:**

1. ✅ Phase 0-4 classification engine complete and tested
2. ✅ USN Journal monitoring detects file events
3. ✅ WebSocket client pushes results to dashboard
4. ✅ Dashboard displays classifications in real-time
5. ✅ Empty file classified as PUBLIC within 2ms
6. ✅ Payroll file classified as RESTRICTED within 50ms
7. ✅ File modification triggers reclassification
8. ✅ Unit tests pass (>90% coverage)
9. ✅ Latency benchmarks meet targets (<50ms p95)
10. ✅ False positive rate <1.5%
11. ✅ False negative rate <1%
12. ✅ Deployed as Windows service
13. ✅ Production monitoring active
14. ✅ No TODOs in code
15. ✅ Code compiles without warnings

**Start coding now. Do not stop until 100% complete.**

---

*PRITRAK V2.0 Real-Time Classification System*  
*Status: PRODUCTION-READY SPECIFICATION*  
*Last Updated: January 11, 2026*
