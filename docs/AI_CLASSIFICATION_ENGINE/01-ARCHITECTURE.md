# 🤖 PRITRAK AI Classification Engine: Complete Architecture Guide
**Version:** 1.0 (Production-Ready)  
**Date:** January 11, 2026  
**Purpose:** Implement enterprise-grade AI-assisted data classification on-premises  
**Integration Target:** DAP (Data Access Platform) DLP System  

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [AI Model Integration](#ai-model-integration)
4. [DLP Synchronization](#dlp-synchronization)
5. [Event Pipeline](#event-pipeline)
6. [Implementation Roadmap](#implementation-roadmap)

---

## Executive Summary

The PRITRAK AI Classification Engine enhances your DAP DLP system with intelligent, on-premises document classification using local language models (LLMs). This eliminates dependency on external APIs while maintaining enterprise security standards.

### Key Objectives
✅ **On-Premises AI** - Run local models, zero data leaves your infrastructure  
✅ **Real-Time Classification** - Classify files within 2-5 seconds  
✅ **DLP Integration** - Classification results populate event logs automatically  
✅ **Multi-Language Support** - English + French with extensible framework  
✅ **Advanced Detection** - NIST/ISO-aligned classification hierarchy  

### Current State (DAP v1.0)
- ✅ File monitoring (create/rename/delete)
- ✅ Event logging with basic metadata
- ✅ Backend event storage (PostgreSQL/SQLite)
- ✅ WebSocket-based frontend updates
- ❌ **Classification layer (TO BE ADDED)**
- ❌ **AI model integration**
- ❌ **Advanced pattern matching**

---

## System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    PRITRAK DLP SYSTEM                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │        FILE SYSTEM EVENT CAPTURE (Agent)             │   │
│  │  - Create, Rename, Delete Events                     │   │
│  │  - File Metadata Extraction                          │   │
│  │  - Path/Size/Extension Analysis                      │   │
│  └────────────────────┬─────────────────────────────────┘   │
│                       │                                      │
│                       ▼                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │      EVENT QUEUE & TRANSMISSION (WebSocket)          │   │
│  │  - Reliable event delivery to backend               │   │
│  │  - Connection pooling & retry logic                 │   │
│  └────────────────────┬─────────────────────────────────┘   │
│                       │                                      │
│                       ▼                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │   🆕 CLASSIFICATION ENGINE (AI Pipeline)             │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │ 1. Pre-Analysis (Lightweight)                │   │   │
│  │  │    - Size check, extension analysis          │   │   │
│  │  │    - Filename heuristics                     │   │   │
│  │  │    - Quick win patterns                      │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │ 2. Content Retrieval & Parsing               │   │   │
│  │  │    - Fetch file content (first 5MB)          │   │   │
│  │  │    - Extract text from documents             │   │   │
│  │  │    - Language detection                      │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │ 3. Keyword/Pattern Matching                  │   │   │
│  │  │    - Ruleset evaluation                      │   │   │
│  │  │    - Regex pattern detection                 │   │   │
│  │  │    - Context-aware scoring                   │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │ 4. AI Model Enhancement (Optional)           │   │   │
│  │  │    - LLM semantic analysis                   │   │   │
│  │  │    - Confidence boosting                     │   │   │
│  │  │    - False positive reduction                │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │ 5. Final Classification Decision             │   │   │
│  │  │    - Classification: PUBLIC/INTERNAL/...    │   │   │
│  │  │    - Confidence score (0-100%)               │   │   │
│  │  │    - Risk level assignment                   │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  └──────────────────┬─────────────────────────────────┘   │
│                     │                                      │
│                     ▼                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  DATABASE STORAGE (Enriched Events)                  │   │
│  │  - Event ID, Timestamp, File Path                    │   │
│  │  - Classification, Confidence, Risk Level            │   │
│  │  - Keyword matches, Pattern detections               │   │
│  │  - Action taken (Allow/Warn/Block)                   │   │
│  └────────────────────┬─────────────────────────────────┘   │
│                       │                                      │
│                       ▼                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FRONTEND EVENT LOG VISUALIZATION                    │   │
│  │  - Real-time event dashboard                         │   │
│  │  - Classification filtering & search                 │   │
│  │  - Risk-based alerting & actions                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 1. **File System Agent** (Existing in DAP)
**Status:** ✅ Complete  
**Responsibility:** Capture file operations  
**Output:** File metadata + event details

```json
{
  "event_id": "evt-20260111-001",
  "timestamp": "2026-01-11T06:15:30Z",
  "event_type": "FILE_CREATED",
  "file_path": "C:\\Finance\\Q1_2026_Forecast.xlsx",
  "file_size": 125440,
  "file_extension": ".xlsx",
  "user": "john.doe",
  "device": "DESKTOP-ABC123"
}
```

#### 2. **Classification Engine** (🆕 To Be Implemented)
**Responsibility:** Enrich events with classification data  
**Processing Time:** 2-5 seconds per file  
**Output:** Classification metadata

```json
{
  "classification": "CONFIDENTIAL",
  "risk_level": "HIGH",
  "confidence": 87.3,
  "detected_keywords": ["forecast", "financial", "Q1"],
  "pattern_matches": [],
  "justification": "Financial planning document with forecasted data",
  "processing_time_ms": 1240
}
```

#### 3. **Database Layer** (Extends Existing)
**Schema Update:** Add classification fields to events table

```sql
ALTER TABLE events ADD COLUMN (
  classification VARCHAR(50),
  confidence_score DECIMAL(5,2),
  risk_level VARCHAR(20),
  detected_keywords TEXT,
  processing_time_ms INTEGER
);
```

#### 4. **Frontend Integration** (Updates Existing UI)
**Enhancement:** Display classification in event log

```html
<!-- Event Row in Dashboard -->
<tr>
  <td>FILE_CREATED</td>
  <td>Q1_2026_Forecast.xlsx</td>
  <td><span class="badge badge-danger">CONFIDENTIAL</span></td>
  <td>87.3%</td>
  <td><span class="tag-high">HIGH</span></td>
</tr>
```

---

## AI Model Integration

### Option 1: Local LLM (Recommended for Security)

**Model:** Ollama + Mistral/Llama 2 (7B parameters)  
**Hardware Requirements:** 
- Minimum: 8GB VRAM GPU or 16GB RAM (CPU inference)
- Recommended: 12GB+ VRAM (inference time: <1 second)

**Workflow:**

```
FILE_EVENT → PRE_ANALYSIS → CONTENT_FETCH → 
  LLM_PROMPT → MODEL_INFERENCE → POST_PROCESS → 
  CLASSIFICATION_RESULT → DATABASE_STORE → UI_UPDATE
```

**Example LLM Prompt (Context-Aware):**

```
You are a data classification expert. Analyze the following document and classify it according to NIST security standards.

DOCUMENT METADATA:
- Filename: Q1_2026_Financial_Forecast.xlsx
- Size: 125KB
- File Type: Excel spreadsheet

DOCUMENT CONTENT (first 2000 chars):
[...document text...]

CLASSIFICATION CATEGORIES:
1. PUBLIC - No security risk
2. INTERNAL - Internal use only, limited disruption if exposed
3. CONFIDENTIAL - Restricted access, significant business impact
4. RESTRICTED - Trade secrets, critical PII, severe consequences

Respond in JSON format:
{
  "classification": "CATEGORY",
  "confidence": 0-100,
  "key_indicators": ["indicator1", "indicator2"],
  "justification": "brief explanation"
}
```

### Option 2: Hybrid Approach (Recommended for Production)

**Tier 1:** Lightweight pattern matching (95% accuracy, <100ms)  
**Tier 2:** LLM enhancement only for borderline cases (5-10% of files)  
**Benefits:** Ultra-fast classification + AI accuracy where needed

```
LIGHTWEIGHT_RULES (100ms)
    ↓
Confidence > 85%? → RETURN RESULT
Confidence 50-85%? → ESCALATE_TO_LLM (1-2 seconds)
    ↓
LLM_ANALYSIS
    ↓
FINAL_RESULT
```

---

## DLP Synchronization

### Event Flow Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  SYNCHRONIZATION PIPELINE                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. EVENT CAPTURE (Agent on Endpoint)                       │
│     └─> FileSystemWatcher catches: CREATE, DELETE, MODIFY  │
│                                                               │
│  2. EVENT TRANSMISSION (WebSocket to Backend)               │
│     └─> Reliable delivery with retry logic                  │
│         └─> Queue management (in-memory + persistence)     │
│                                                               │
│  3. CLASSIFICATION REQUEST (Backend Queue Consumer)          │
│     └─> Dequeue event                                        │
│     └─> Check cache (identical files)                        │
│     └─> Route to classification engine                       │
│                                                               │
│  4. CLASSIFICATION PROCESSING (Multi-stage Pipeline)         │
│     ├─> Stage 1: Pre-Analysis (extension, size, filename)  │
│     ├─> Stage 2: Content Parsing (extract text/metadata)   │
│     ├─> Stage 3: Pattern Matching (keywords + regex)        │
│     ├─> Stage 4: AI Enhancement (if confidence < 85%)      │
│     └─> Stage 5: Decision (final classification)            │
│                                                               │
│  5. ENRICHMENT (Add Classification to Event)                │
│     └─> {event + classification_result + metadata}          │
│                                                               │
│  6. STORAGE (Persist Enriched Event)                        │
│     └─> Events table (PostgreSQL/SQLite)                    │
│     └─> Update incident records if HIGH risk                │
│                                                               │
│  7. FRONTEND SYNC (WebSocket Broadcast)                     │
│     └─> Notify connected dashboard clients                  │
│     └─> Update in real-time with classification             │
│                                                               │
│  8. ACTION EXECUTION (Policy-Based Rules)                   │
│     ├─> RESTRICTED + Transfer attempt? → BLOCK             │
│     ├─> CONFIDENTIAL + USB device? → WARN + LOG            │
│     ├─> HIGH confidence match? → ESCALATE to SOC           │
│     └─> Store action in audit log                           │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Data Structures for Sync

**Event Record (After Classification):**

```go
type ClassifiedEvent struct {
    // Original event data
    EventID         string    `json:"event_id"`
    Timestamp       time.Time `json:"timestamp"`
    DeviceID        string    `json:"device_id"`
    FilePath        string    `json:"file_path"`
    FileSize        int64     `json:"file_size"`
    FileExtension   string    `json:"file_extension"`
    EventType       string    `json:"event_type"` // CREATE, DELETE, MODIFY
    
    // Classification data
    Classification  string    `json:"classification"` // PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
    ConfidenceScore float32   `json:"confidence_score"` // 0-100
    RiskLevel       string    `json:"risk_level"` // LOW, MEDIUM, HIGH, CRITICAL
    Justification   string    `json:"justification"`
    
    // Detection details
    DetectedKeywords []string `json:"detected_keywords"`
    PatternMatches   []string `json:"pattern_matches"`
    Language         string   `json:"language"` // "en", "fr"
    
    // Performance metrics
    ProcessingTimeMs int      `json:"processing_time_ms"`
    
    // Action taken
    Action          string    `json:"action"` // ALLOW, WARN, BLOCK
    ActionReason    string    `json:"action_reason"`
}
```

### Synchronization Guarantees

| Guarantee | Implementation |
|-----------|-----------------|
| **Ordering** | Process events in sequence per device |
| **Exactly-Once** | Idempotent classification + event deduplication |
| **Latency** | < 5 seconds end-to-end (including AI) |
| **Durability** | Persist event before UI update |
| **Consistency** | All replicas see same classification result |

---

## Event Pipeline

### Detailed Processing Stages

#### Stage 1: Pre-Analysis (Lightweight, <100ms)

```go
// File: backend/internal/classification/pre_analyzer.go
type PreAnalysisResult struct {
    Classification string  // Quick classification if evident
    QuickWin       bool    // true = don't need further analysis
    Confidence     float32
    Justification  string
}

func PreAnalyze(fileInfo *FileInfo) *PreAnalysisResult {
    // 1. Size check
    if fileInfo.Size == 0 || (fileInfo.Size < 100 && onlyWhitespace) {
        return &PreAnalysisResult{
            Classification: "PUBLIC",
            QuickWin: true,
            Confidence: 100,
            Justification: "Empty file",
        }
    }
    
    // 2. Extension check for known sensitive files
    sensitiveExts := []string{".key", ".pem", ".p12", ".pfx", ".db", ".sql"}
    if contains(sensitiveExts, fileInfo.Extension) && fileInfo.Size > 1024 {
        return &PreAnalysisResult{
            Classification: "RESTRICTED",
            QuickWin: true,
            Confidence: 95,
            Justification: fmt.Sprintf("Sensitive file type: %s", fileInfo.Extension),
        }
    }
    
    // 3. Filename analysis
    if contains(fileInfo.Name, []string{"password", "secret", "api_key"}) {
        return &PreAnalysisResult{
            Classification: "RESTRICTED",
            QuickWin: true,
            Confidence: 98,
            Justification: "Sensitive filename pattern detected",
        }
    }
    
    return nil // Continue to Stage 2
}
```

#### Stage 2: Content Parsing (1-2 seconds)

```go
// File: backend/internal/classification/content_parser.go
type ParsedContent struct {
    RawText      string
    Language     string   // "en", "fr", "mixed"
    WordCount    int
    KeyPhrases   []string
    Metadata     map[string]string
}

func ParseContent(filePath string, maxBytes int64) (*ParsedContent, error) {
    // 1. Read file (limit to 5MB to avoid memory issues)
    content, err := ioutil.ReadFile(filePath)
    if err != nil {
        return nil, err
    }
    
    if int64(len(content)) > maxBytes {
        content = content[:maxBytes]
    }
    
    // 2. Detect encoding and convert to UTF-8
    detected := chardet.NewTextDetector().DetectBest(content)
    
    // 3. Extract text based on file type
    var text string
    switch filepath.Ext(filePath) {
    case ".pdf":
        text, _ = extractPDF(content)
    case ".docx":
        text, _ = extractDOCX(content)
    case ".xlsx":
        text, _ = extractXLSX(content)
    default:
        text = string(content)
    }
    
    // 4. Language detection
    langDetector := whichlang.NewDetector()
    langs := langDetector.DetectLangs(text)
    lang := langs[0].Lang().String() // "en", "fr", etc.
    
    return &ParsedContent{
        RawText:    text,
        Language:   lang,
        WordCount:  len(strings.Fields(text)),
        KeyPhrases: extractKeyPhrases(text),
        Metadata:   extractMetadata(filePath),
    }, nil
}
```

#### Stage 3: Pattern Matching (1-2 seconds)

```go
// File: backend/internal/classification/matcher.go
type ScoringResult struct {
    ClassificationScores map[string]float32 // PUBLIC: 0.2, INTERNAL: 1.5, ...
    DetectedKeywords     []string
    RegexMatches         []RegexMatch
    ConfidenceBoost      float32
}

func MatchPatterns(content *ParsedContent) *ScoringResult {
    result := &ScoringResult{
        ClassificationScores: make(map[string]float32),
        DetectedKeywords:     []string{},
        RegexMatches:         []RegexMatch{},
    }
    
    // 1. Load keyword lists for detected language
    keywords := loadKeywords(content.Language)
    
    // 2. Scan for keywords with context
    for classification, kwList := range keywords {
        for _, kw := range kwList {
            matches := countKeywordOccurrences(content.RawText, kw)
            if matches > 0 {
                // Weight by classification importance
                weight := getKeywordWeight(classification)
                contextBoost := getContextBoost(content.RawText, kw)
                result.ClassificationScores[classification] += float32(matches) * weight * contextBoost
                result.DetectedKeywords = append(result.DetectedKeywords, kw)
            }
        }
    }
    
    // 3. Run regex patterns
    patterns := loadRegexPatterns() // Credit cards, SSNs, API keys, etc.
    for patternName, regex := range patterns {
        matches := regex.FindAllString(content.RawText, -1)
        if len(matches) > 0 {
            result.RegexMatches = append(result.RegexMatches, RegexMatch{
                Pattern: patternName,
                Count:   len(matches),
            })
            
            // Sensitive data detected = boost RESTRICTED score
            if isSensitivePattern(patternName) {
                result.ClassificationScores["RESTRICTED"] += 3.0 * float32(len(matches))
            }
        }
    }
    
    return result
}
```

#### Stage 4: AI Enhancement (Optional, 1-2 seconds)

```go
// File: backend/internal/classification/ai_enhancer.go
type AIEnhancementResult struct {
    Classification string
    Confidence     float32
    Reasoning      string
}

func EnhanceWithAI(content *ParsedContent, scoringResult *ScoringResult) (*AIEnhancementResult, error) {
    // Only call LLM if confidence < 85% (fast path for obvious cases)
    currentBest := findHighestScore(scoringResult.ClassificationScores)
    if currentBest.score > 85.0 {
        return nil // High confidence, skip LLM
    }
    
    // Build prompt
    prompt := buildClassificationPrompt(content, scoringResult)
    
    // Call local LLM (Ollama)
    client := ollama.NewClient("http://localhost:11434")
    response, err := client.Generate(context.Background(), &ollama.GenerateRequest{
        Model:  "mistral",
        Prompt: prompt,
        Stream: false,
    })
    if err != nil {
        // Fallback to rule-based if LLM fails
        return fallbackClassification(scoringResult), nil
    }
    
    // Parse LLM response
    result := parseAIResponse(response.Response)
    return result, nil
}
```

#### Stage 5: Final Decision (100ms)

```go
// File: backend/internal/classification/decision_engine.go
type FinalClassificationDecision struct {
    Classification string  // PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
    Confidence     float32 // 0-100
    RiskLevel      string  // LOW, MEDIUM, HIGH, CRITICAL
    Justification  string
}

func MakeDecision(context *ClassificationContext) *FinalClassificationDecision {
    // 1. Apply thresholds
    thresholds := map[string]float32{
        "RESTRICTED":   8.5,
        "CONFIDENTIAL": 5.5,
        "INTERNAL":     2.0,
        "PUBLIC":       0.5,
    }
    
    // 2. Find winning classification
    bestClass := ""
    bestScore := 0.0
    for class, score := range context.ScoringResult.ClassificationScores {
        if score > bestScore && score >= thresholds[class] {
            bestScore = score
            bestClass = class
        }
    }
    
    // 3. Handle edge cases
    if bestClass == "" {
        bestClass = "INTERNAL" // Conservative default
    }
    
    // 4. Calculate normalized confidence
    confidence := normalizeScore(bestScore, thresholds[bestClass]) * 100
    
    // 5. Map to risk level
    riskMap := map[string]string{
        "PUBLIC":       "LOW",
        "INTERNAL":     "LOW",
        "CONFIDENTIAL": "HIGH",
        "RESTRICTED":   "CRITICAL",
    }
    
    return &FinalClassificationDecision{
        Classification: bestClass,
        Confidence:     confidence,
        RiskLevel:      riskMap[bestClass],
        Justification:  buildJustification(context),
    }
}
```

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [ ] Create classification module structure
- [ ] Implement pre-analysis engine
- [ ] Build content parser (text/PDF/Excel)
- [ ] Set up keyword/regex pattern database
- [ ] Integrate with existing event storage

### Phase 2: Core Classification (Weeks 3-4)
- [ ] Pattern matching engine
- [ ] Scoring algorithm
- [ ] Database schema updates
- [ ] Backend API endpoints for classification
- [ ] Unit tests (>85% coverage)

### Phase 3: AI Enhancement (Weeks 5-6)
- [ ] Ollama integration setup
- [ ] LLM model fine-tuning (optional)
- [ ] AI prompt engineering
- [ ] Fallback mechanisms
- [ ] Performance optimization

### Phase 4: Integration (Weeks 7-8)
- [ ] Event pipeline synchronization
- [ ] Frontend UI updates
- [ ] Real-time WebSocket updates
- [ ] Performance testing (2000+ files/hour)
- [ ] Security audit

### Phase 5: Production (Weeks 9-10)
- [ ] Load testing (100+ concurrent agents)
- [ ] Disaster recovery procedures
- [ ] Monitoring & alerting setup
- [ ] Documentation & training
- [ ] Beta testing with pilot users

---

## Next Steps

Continue reading in:
- **[02-IMPLEMENTATION.md](./02-IMPLEMENTATION.md)** - Detailed code implementation
- **[03-AI-SETUP.md](./03-AI-SETUP.md)** - LLM configuration and tuning
- **[04-INTEGRATION.md](./04-INTEGRATION.md)** - DLP synchronization details
- **[05-DEPLOYMENT.md](./05-DEPLOYMENT.md)** - Production deployment guide

---

**Last Updated:** January 11, 2026  
**Maintained By:** PRITRAK Security Team  
**Status:** 🟢 Active Development

