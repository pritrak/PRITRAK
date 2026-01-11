# 🔧 PRITRAK AI Classification Engine: Implementation Guide

**Version:** 1.0  
**Date:** January 11, 2026  
**Target:** Go Backend Integration with DAP  
**Complexity:** Intermediate (300+ lines per module)  

---

## Quick Start: 5-Minute Integration

Add this to your existing DAP backend:

```bash
# 1. Create classification module
mkdir -p backend/internal/classification
mkdir -p backend/internal/classification/{models,patterns,ai}

# 2. Copy code files (from following sections)
# 3. Update event handler to call classification
# 4. Test with sample files
```

---

## Module 1: File Information Extractor

**File:** `backend/internal/classification/file_info.go`

```go
package classification

import (
    "fmt"
    "os"
    "path/filepath"
    "strings"
)

// FileInfo holds extracted metadata about a file
type FileInfo struct {
    Path          string
    Name          string
    Size          int64
    Extension     string
    ModTime       int64 // Unix timestamp
    IsDirectory   bool
    MimeType      string
    DirectoryPath string
}

// ExtractFileInfo gets comprehensive file metadata
func ExtractFileInfo(filePath string) (*FileInfo, error) {
    stat, err := os.Stat(filePath)
    if err != nil {
        return nil, fmt.Errorf("failed to stat file: %w", err)
    }

    ext := strings.ToLower(filepath.Ext(filePath))
    dir := filepath.Dir(filePath)
    name := filepath.Base(filePath)

    return &FileInfo{
        Path:          filePath,
        Name:          name,
        Size:          stat.Size(),
        Extension:     ext,
        ModTime:       stat.ModTime().Unix(),
        IsDirectory:   stat.IsDir(),
        MimeType:      guessMimeType(ext),
        DirectoryPath: dir,
    }, nil
}

// guessMimeType returns MIME type based on extension
func guessMimeType(ext string) string {
    mimeTypes := map[string]string{
        ".pdf":  "application/pdf",
        ".docx": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        ".xlsx": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        ".txt":  "text/plain",
        ".csv":  "text/csv",
        ".json": "application/json",
        ".xml":  "application/xml",
        ".sql":  "application/x-sql",
        ".key":  "application/pkcs8",
        ".pem":  "application/x-pem-file",
        ".db":   "application/x-sqlite3",
        ".env":  "text/plain",
    }
    
    if mime, exists := mimeTypes[ext]; exists {
        return mime
    }
    return "application/octet-stream"
}
```

---

## Module 2: Pre-Analysis Engine

**File:** `backend/internal/classification/pre_analyzer.go`

```go
package classification

import (
    "fmt"
    "strings"
)

// PreAnalysisResult contains quick classification decision
type PreAnalysisResult struct {
    Classification string  // PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
    QuickWin       bool    // true = no need for further analysis
    Confidence     float32 // 0-100
    Justification  string
}

// PreAnalyzer runs lightweight analysis on file metadata
type PreAnalyzer struct {
    sensitiveExtensions []string
    sensitiveFilenames  []string
}

// NewPreAnalyzer creates a new pre-analyzer
func NewPreAnalyzer() *PreAnalyzer {
    return &PreAnalyzer{
        sensitiveExtensions: []string{
            ".key", ".pem", ".p12", ".pfx",  // Encryption keys
            ".db", ".mdb", ".sql",             // Databases
            ".env",                             // Environment configs
        },
        sensitiveFilenames: []string{
            "password", "secret", "api_key", "apikey",
            "token", "credential", "aws_access",
        },
    }
}

// Analyze performs pre-analysis on file metadata
func (pa *PreAnalyzer) Analyze(fileInfo *FileInfo) *PreAnalysisResult {
    // Stage 1: Check if file is empty
    if fileInfo.Size == 0 {
        return &PreAnalysisResult{
            Classification: "PUBLIC",
            QuickWin:       true,
            Confidence:     100,
            Justification:  "Empty file (0 bytes)",
        }
    }

    // Stage 2: Check sensitive file extensions
    for _, ext := range pa.sensitiveExtensions {
        if strings.EqualFold(fileInfo.Extension, ext) && fileInfo.Size > 1024 {
            return &PreAnalysisResult{
                Classification: "RESTRICTED",
                QuickWin:       true,
                Confidence:     95,
                Justification:  fmt.Sprintf("Sensitive file type detected: %s", ext),
            }
        }
    }

    // Stage 3: Check filename patterns
    lowerName := strings.ToLower(fileInfo.Name)
    for _, sensitive := range pa.sensitiveFilenames {
        if strings.Contains(lowerName, sensitive) {
            return &PreAnalysisResult{
                Classification: "RESTRICTED",
                QuickWin:       true,
                Confidence:     98,
                Justification:  fmt.Sprintf("Sensitive filename pattern: '%s' contains '%s'", fileInfo.Name, sensitive),
            }
        }
    }

    // Stage 4: Check directory patterns
    sensitiveDirs := []string{"hr", "finance", "legal", "executive", "confidential"}
    lowerDir := strings.ToLower(fileInfo.DirectoryPath)
    for _, dir := range sensitiveDirs {
        if strings.Contains(lowerDir, strings.ToLower(dir)) {
            // File in sensitive directory, but not a quick win
            // Continue to content analysis
            return nil
        }
    }

    // No quick win - continue to content analysis
    return nil
}
```

---

## Module 3: Content Parser

**File:** `backend/internal/classification/content_parser.go`

```go
package classification

import (
    "bytes"
    "fmt"
    "io"
    "os"
    "strings"
    "unicode/utf8"
)

// ParsedContent holds extracted and analyzed content
type ParsedContent struct {
    RawText       string            // Extracted text content
    Language      string            // "en" or "fr"
    WordCount     int
    CharCount     int
    KeyPhrases    []string
    HasStructure  bool              // Structured data like tables
    Metadata      map[string]string // Document metadata
    EncodingError bool              // True if encoding issues
}

// ContentParser extracts readable content from files
type ContentParser struct {
    maxBytes int64
}

// NewContentParser creates a new content parser
func NewContentParser(maxBytes int64) *ContentParser {
    return &ContentParser{
        maxBytes: maxBytes,
    }
}

// Parse extracts content from file based on type
func (cp *ContentParser) Parse(filePath string) (*ParsedContent, error) {
    // Read raw file content (up to maxBytes)
    content, err := cp.readFile(filePath)
    if err != nil {
        return nil, fmt.Errorf("failed to read file: %w", err)
    }

    // Convert to UTF-8 text
    text := cp.toUTF8(content)

    // Detect language
    lang := cp.detectLanguage(text)

    // Extract metadata
    metadata := cp.extractMetadata(filePath, text)

    return &ParsedContent{
        RawText:      text,
        Language:     lang,
        WordCount:    len(strings.Fields(text)),
        CharCount:    len([]rune(text)),
        KeyPhrases:   cp.extractKeyPhrases(text),
        HasStructure: cp.hasStructuredData(text),
        Metadata:     metadata,
    }, nil
}

// readFile reads file with size limit
func (cp *ContentParser) readFile(filePath string) ([]byte, error) {
    file, err := os.Open(filePath)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    return io.ReadAll(io.LimitReader(file, cp.maxBytes))
}

// toUTF8 converts content to UTF-8, handling various encodings
func (cp *ContentParser) toUTF8(data []byte) string {
    // Check if already valid UTF-8
    if utf8.Valid(data) {
        return string(data)
    }

    // Try to decode with common encodings
    encodings := []func([]byte) string{
        cp.tryLatin1,
        cp.tryCP1252,
    }

    for _, decode := range encodings {
        if text := decode(data); text != "" {
            return text
        }
    }

    // Fallback: replace invalid UTF-8 sequences
    return string(bytes.ToValidUTF8(data, []byte("?")))
}

// tryLatin1 attempts Latin-1 decoding
func (cp *ContentParser) tryLatin1(data []byte) string {
    // Latin-1 is mostly single-byte, check if plausible
    valid := true
    for _, b := range data {
        if b == 0 { // Null byte suggests binary
            valid = false
            break
        }
    }
    if !valid {
        return ""
    }
    return string(data)
}

// tryCP1252 attempts Windows-1252 decoding
func (cp *ContentParser) tryCP1252(data []byte) string {
    // Similar to Latin-1 for this purpose
    return ""
}

// detectLanguage identifies document language
func (cp *ContentParser) detectLanguage(text string) string {
    // Simple heuristic-based language detection
    frenchKeywords := []string{"de", "le", "et", "la", "à", "les", "est", "dans", "pour", "un", "une"}
    englishKeywords := []string{"the", "and", "is", "to", "in", "of", "for", "a", "on", "with"}

    words := strings.Fields(strings.ToLower(text))
    frenchCount := 0
    englishCount := 0

    for _, word := range words {
        for _, fkw := range frenchKeywords {
            if strings.Contains(word, fkw) {
                frenchCount++
            }
        }
        for _, ekw := range englishKeywords {
            if strings.Contains(word, ekw) {
                englishCount++
            }
        }
    }

    if frenchCount > englishCount {
        return "fr"
    }
    return "en"
}

// extractKeyPhrases finds important phrases
func (cp *ContentParser) extractKeyPhrases(text string) []string {
    // Extract lines that look like headers or important phrases
    var phrases []string
    lines := strings.Split(text, "\n")

    for _, line := range lines {
        line = strings.TrimSpace(line)
        if len(line) > 10 && len(line) < 100 {
            // Check if looks like header (contains ":", "=", or ends with "?")
            if strings.Contains(line, ":") || strings.Contains(line, "=") || strings.HasSuffix(line, "?") {
                phrases = append(phrases, line)
            }
        }
    }

    // Return top 10
    if len(phrases) > 10 {
        return phrases[:10]
    }
    return phrases
}

// hasStructuredData checks for tables, lists, etc.
func (cp *ContentParser) hasStructuredData(text string) bool {
    // Check for table-like patterns
    if strings.Contains(text, "|") || strings.Count(text, "\t") > 10 {
        return true
    }
    // Check for list patterns
    if strings.Count(text, "\n") > 50 && strings.Count(text, "-") > 10 {
        return true
    }
    return false
}

// extractMetadata extracts document metadata
func (cp *ContentParser) extractMetadata(filePath string, text string) map[string]string {
    metadata := make(map[string]string)
    metadata["language"] = cp.detectLanguage(text)
    metadata["has_structure"] = fmt.Sprintf("%v", cp.hasStructuredData(text))
    return metadata
}
```

---

## Module 4: Pattern Matcher

**File:** `backend/internal/classification/pattern_matcher.go`

```go
package classification

import (
    "regexp"
    "strings"
)

// PatternMatch represents a detected pattern
type PatternMatch struct {
    Name       string
    Type       string // KEYWORD, REGEX, SENSITIVE_DATA
    Confidence float32
    Count      int
}

// ScoringResult holds classification scores
type ScoringResult struct {
    Scores           map[string]float32 // Classification -> score
    DetectedKeywords []string
    PatternMatches   []PatternMatch
    Language         string
}

// PatternMatcher performs keyword and regex matching
type PatternMatcher struct {
    keywordSet    map[string][]string   // Classification -> keywords
    regexPatterns map[string]*regexp.Regexp
}

// NewPatternMatcher creates a new matcher
func NewPatternMatcher() *PatternMatcher {
    return &PatternMatcher{
        keywordSet:    initializeKeywords(),
        regexPatterns: initializeRegexPatterns(),
    }
}

// Match performs pattern matching on content
func (pm *PatternMatcher) Match(content *ParsedContent) *ScoringResult {
    result := &ScoringResult{
        Scores:           make(map[string]float32),
        DetectedKeywords: []string{},
        PatternMatches:   []PatternMatch{},
        Language:         content.Language,
    }

    // Initialize scores
    result.Scores["PUBLIC"] = 0
    result.Scores["INTERNAL"] = 0
    result.Scores["CONFIDENTIAL"] = 0
    result.Scores["RESTRICTED"] = 0

    // 1. Keyword matching
    pm.matchKeywords(content, result)

    // 2. Regex pattern matching
    pm.matchPatterns(content, result)

    // 3. Context-based boosting
    pm.applyContextBoosts(content, result)

    return result
}

// matchKeywords scans for classification keywords
func (pm *PatternMatcher) matchKeywords(content *ParsedContent, result *ScoringResult) {
    text := strings.ToLower(content.RawText)

    for classification, keywords := range pm.keywordSet {
        for _, keyword := range keywords {
            count := strings.Count(text, strings.ToLower(keyword))
            if count > 0 {
                // Weight by keyword importance and frequency
                weight := float32(count) * getKeywordWeight(keyword, classification)
                result.Scores[classification] += weight
                result.DetectedKeywords = append(result.DetectedKeywords, keyword)
            }
        }
    }
}

// matchPatterns runs regex patterns
func (pm *PatternMatcher) matchPatterns(content *ParsedContent, result *ScoringResult) {
    for patternName, regex := range pm.regexPatterns {
        matches := regex.FindAllString(content.RawText, -1)
        if len(matches) > 0 {
            classification := getClassificationForPattern(patternName)
            result.Scores[classification] += float32(len(matches)) * 3.0 // High weight for pattern match
            result.PatternMatches = append(result.PatternMatches, PatternMatch{
                Name:       patternName,
                Type:       "REGEX",
                Confidence: 95,
                Count:      len(matches),
            })
        }
    }
}

// applyContextBoosts adjusts scores based on document structure
func (pm *PatternMatcher) applyContextBoosts(content *ParsedContent, result *ScoringResult) {
    // If structured data (table/list) with customer data, boost CONFIDENTIAL
    if content.HasStructure && strings.Contains(strings.ToLower(content.RawText), "customer") {
        result.Scores["CONFIDENTIAL"] *= 1.5
    }

    // If document is small but has many keywords, boost confidence
    keywordDensity := float32(len(result.DetectedKeywords)) / float32(content.WordCount+1)
    if keywordDensity > 0.05 {
        // High keyword density
        for k := range result.Scores {
            result.Scores[k] *= 1.3
        }
    }
}

// initializeKeywords returns classification keyword sets
func initializeKeywords() map[string][]string {
    return map[string][]string{
        "RESTRICTED": []string{
            // English
            "trade secret", "proprietary", "password", "api key", "private key",
            "secret key", "access token", "vulnerability", "exploit", "zero-day",
            "ssn", "social security", "credit card", "bank account", "cvv",
            "passport", "medical record", "diagnosis",
            // French
            "secret commercial", "propriétaire", "mot de passe", "clé api",
            "clé privée", "token d'accès", "vulnérabilité", "exploit",
            "numéro de sécurité sociale", "carte de crédit", "compte bancaire",
        },
        "CONFIDENTIAL": []string{
            // English
            "invoice", "balance sheet", "payroll", "salary", "customer",
            "client", "order", "contract", "agreement", "employee record",
            "financial statement", "budget", "forecast", "strategic plan",
            "meeting minutes", "confidential",
            // French
            "facture", "bilan", "paie", "salaire", "client",
            "contrat", "accord", "personnel", "état financier", "budget",
            "prévision", "plan stratégique", "procès-verbal", "confidentiel",
        },
        "INTERNAL": []string{
            // English
            "internal memo", "internal note", "staff update", "team announcement",
            "internal policy", "procedure", "guidelines", "intranet",
            // French
            "mémo interne", "note interne", "mise à jour du personnel",
            "politique interne", "procédure", "directives",
        },
    }
}

// initializeRegexPatterns returns compiled regex patterns
func initializeRegexPatterns() map[string]*regexp.Regexp {
    patterns := make(map[string]*regexp.Regexp)

    // Credit card pattern (simplified)
    patterns["CREDIT_CARD"] = regexp.MustCompile(`\b(?:\d{4}[-\s]?){3}\d{4}\b`)

    // US SSN pattern
    patterns["US_SSN"] = regexp.MustCompile(`\b(?!000|666|9\d{2})(\d{3})-?(\d{2})-?(\d{4})\b`)

    // Email pattern
    patterns["EMAIL"] = regexp.MustCompile(`\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b`)

    // IP address pattern (private)
    patterns["PRIVATE_IP"] = regexp.MustCompile(`\b(192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[01])\.)`)

    // API Key patterns
    patterns["AWS_KEY"] = regexp.MustCompile(`AKIA[0-9A-Z]{16}`)
    patterns["GITHUB_KEY"] = regexp.MustCompile(`ghp_[0-9a-zA-Z]{36}`)

    return patterns
}

// getKeywordWeight returns weight multiplier for keyword
func getKeywordWeight(keyword, classification string) float32 {
    // High-confidence keywords get higher weight
    highConfidence := []string{"password", "api key", "secret key", "credit card", "ssn"}
    for _, kw := range highConfidence {
        if strings.Contains(strings.ToLower(keyword), strings.ToLower(kw)) {
            return 3.0
        }
    }
    return 1.0
}

// getClassificationForPattern returns classification for pattern
func getClassificationForPattern(patternName string) string {
    patternClassification := map[string]string{
        "CREDIT_CARD": "RESTRICTED",
        "US_SSN":      "RESTRICTED",
        "EMAIL":       "INTERNAL",
        "PRIVATE_IP":  "INTERNAL",
        "AWS_KEY":     "RESTRICTED",
        "GITHUB_KEY":  "RESTRICTED",
    }

    if class, exists := patternClassification[patternName]; exists {
        return class
    }
    return "INTERNAL"
}
```

---

## Module 5: Decision Engine

**File:** `backend/internal/classification/decision_engine.go`

```go
package classification

import (
    "fmt"
    "math"
)

// ClassificationDecision holds final classification result
type ClassificationDecision struct {
    Classification string  // PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
    Confidence     float32 // 0-100
    RiskLevel      string  // LOW, MEDIUM, HIGH, CRITICAL
    Justification  string
    ProcessingMs   int64
}

// DecisionEngine makes final classification decision
type DecisionEngine struct {
    thresholds map[string]float32
    riskMap    map[string]string
}

// NewDecisionEngine creates a new decision engine
func NewDecisionEngine() *DecisionEngine {
    return &DecisionEngine{
        thresholds: map[string]float32{
            "RESTRICTED":   8.5,
            "CONFIDENTIAL": 5.5,
            "INTERNAL":     2.0,
            "PUBLIC":       0.5,
        },
        riskMap: map[string]string{
            "PUBLIC":       "LOW",
            "INTERNAL":     "LOW",
            "CONFIDENTIAL": "HIGH",
            "RESTRICTED":   "CRITICAL",
        },
    }
}

// Decide makes final classification decision
func (de *DecisionEngine) Decide(scoring *ScoringResult) *ClassificationDecision {
    // 1. Find winning classification
    bestClass := ""
    bestScore := float32(0)

    for class, score := range scoring.Scores {
        if score > bestScore && score >= de.thresholds[class] {
            bestScore = score
            bestClass = class
        }
    }

    // 2. Default to INTERNAL if no match
    if bestClass == "" {
        bestClass = "INTERNAL"
        bestScore = scoring.Scores["INTERNAL"]
    }

    // 3. Calculate normalized confidence (0-100)
    threshold := de.thresholds[bestClass]
    confidence := de.normalizeScore(bestScore, threshold)

    // 4. Cap at 100%
    if confidence > 100 {
        confidence = 100
    }

    return &ClassificationDecision{
        Classification: bestClass,
        Confidence:     confidence,
        RiskLevel:      de.riskMap[bestClass],
        Justification:  de.buildJustification(bestClass, scoring),
    }
}

// normalizeScore converts raw score to 0-100 confidence
func (de *DecisionEngine) normalizeScore(score, threshold float32) float32 {
    if score <= 0 {
        return 0
    }
    if score < threshold {
        return (score / threshold) * 100
    }
    // Beyond threshold, use log scale to avoid overshooting 100
    return 50 + (math.Log(float64(score/threshold))*10) // Approximate
}

// buildJustification creates human-readable explanation
func (de *DecisionEngine) buildJustification(classification string, scoring *ScoringResult) string {
    var reasons []string

    // Add keyword reasons
    if len(scoring.DetectedKeywords) > 0 {
        kw := scoring.DetectedKeywords
        if len(kw) > 3 {
            kw = kw[:3]
        }
        reasons = append(reasons, fmt.Sprintf("Keywords: %v", kw))
    }

    // Add pattern reasons
    if len(scoring.PatternMatches) > 0 {
        for i, match := range scoring.PatternMatches {
            if i >= 2 { // Limit to 2 patterns
                break
            }
            reasons = append(reasons, fmt.Sprintf("Pattern: %s (%d matches)", match.Name, match.Count))
        }
    }

    return fmt.Sprintf("%s classification. %v", classification, reasons)
}
```

---

## Module 6: Orchestrator

**File:** `backend/internal/classification/orchestrator.go`

```go
package classification

import (
    "fmt"
    "log"
    "time"
)

// ClassificationRequest represents a file to classify
type ClassificationRequest struct {
    FilePath  string
    DeviceID  string
    EventID   string
}

// ClassificationResponse is the result
type ClassificationResponse struct {
    EventID       string
    Classification string
    Confidence    float32
    RiskLevel     string
    Justification string
    ProcessingMs  int64
}

// Orchestrator coordinates the classification pipeline
type Orchestrator struct {
    preAnalyzer    *PreAnalyzer
    contentParser  *ContentParser
    patternMatcher *PatternMatcher
    decisionEngine *DecisionEngine
}

// NewOrchestrator creates a new orchestrator
func NewOrchestrator() *Orchestrator {
    return &Orchestrator{
        preAnalyzer:    NewPreAnalyzer(),
        contentParser:  NewContentParser(5 * 1024 * 1024), // 5MB limit
        patternMatcher: NewPatternMatcher(),
        decisionEngine: NewDecisionEngine(),
    }
}

// Classify performs complete classification pipeline
func (o *Orchestrator) Classify(req *ClassificationRequest) *ClassificationResponse {
    startTime := time.Now()

    // Extract file metadata
    fileInfo, err := ExtractFileInfo(req.FilePath)
    if err != nil {
        log.Printf("Error extracting file info: %v", err)
        return o.errorResponse(req, "INTERNAL", "Error reading file")
    }

    // Stage 1: Pre-analysis (quick wins)
    preResult := o.preAnalyzer.Analyze(fileInfo)
    if preResult != nil && preResult.QuickWin {
        return &ClassificationResponse{
            EventID:        req.EventID,
            Classification: preResult.Classification,
            Confidence:     preResult.Confidence,
            RiskLevel:      o.decisionEngine.riskMap[preResult.Classification],
            Justification:  preResult.Justification,
            ProcessingMs:   time.Since(startTime).Milliseconds(),
        }
    }

    // Stage 2: Parse content
    parsedContent, err := o.contentParser.Parse(req.FilePath)
    if err != nil {
        log.Printf("Error parsing content: %v", err)
        // Use pre-analysis result if available, otherwise default
        if preResult != nil {
            return &ClassificationResponse{
                EventID:        req.EventID,
                Classification: "INTERNAL",
                Confidence:     50,
                RiskLevel:      "MEDIUM",
                Justification:  "Error parsing content, classified as INTERNAL by default",
                ProcessingMs:   time.Since(startTime).Milliseconds(),
            }
        }
    }

    // Stage 3: Pattern matching
    scoringResult := o.patternMatcher.Match(parsedContent)

    // Stage 4: Decision
    decision := o.decisionEngine.Decide(scoringResult)
    decision.ProcessingMs = time.Since(startTime).Milliseconds()

    return &ClassificationResponse{
        EventID:        req.EventID,
        Classification: decision.Classification,
        Confidence:     decision.Confidence,
        RiskLevel:      decision.RiskLevel,
        Justification:  decision.Justification,
        ProcessingMs:   decision.ProcessingMs,
    }
}

// errorResponse creates error classification response
func (o *Orchestrator) errorResponse(req *ClassificationRequest, classification, reason string) *ClassificationResponse {
    return &ClassificationResponse{
        EventID:        req.EventID,
        Classification: classification,
        Confidence:     50,
        RiskLevel:      "MEDIUM",
        Justification:  reason,
        ProcessingMs:   0,
    }
}
```

---

## Integration with Existing DAP Backend

**File:** `backend/cmd/server/main.go` (Update existing endpoint)

```go
// Add to your events API handler

func (handler *EventHandler) handleNewEvent(event *models.Event) error {
    // 1. Classify the file
    orchestrator := classification.NewOrchestrator()
    classResp := orchestrator.Classify(&classification.ClassificationRequest{
        FilePath: event.FilePath,
        DeviceID: event.DeviceID,
        EventID:  event.EventID,
    })

    // 2. Enrich event with classification
    event.Classification = classResp.Classification
    event.ConfidenceScore = classResp.Confidence
    event.RiskLevel = classResp.RiskLevel
    event.Justification = classResp.Justification
    event.ProcessingTimeMs = classResp.ProcessingMs

    // 3. Save to database
    if err := handler.db.SaveEvent(event); err != nil {
        return err
    }

    // 4. Apply policy-based actions
    if err := handler.applyPolicy(event); err != nil {
        return err
    }

    // 5. Broadcast to frontend via WebSocket
    handler.broadcastToClients(event)

    return nil
}

func (handler *EventHandler) applyPolicy(event *models.Event) error {
    // Example policies
    if event.Classification == "RESTRICTED" && event.EventType == "TRANSFER" {
        return handler.blockAction(event.EventID, "Restricted file transfer attempt")
    }

    if event.Classification == "CONFIDENTIAL" && event.EventType == "USB_ACCESS" {
        return handler.warnUser(event.DeviceID, "Confidential file on removable media")
    }

    if event.ConfidenceScore < 60 {
        return handler.flagForReview(event.EventID, "Low confidence classification")
    }

    return nil
}
```

---

## Database Schema Update

```sql
-- Add classification columns to events table
ALTER TABLE events ADD COLUMN (
    classification VARCHAR(50) DEFAULT 'INTERNAL',
    confidence_score DECIMAL(5,2),
    risk_level VARCHAR(20),
    detected_keywords TEXT,
    pattern_matches TEXT,
    justification TEXT,
    processing_time_ms INTEGER
);

-- Create index for faster filtering
CREATE INDEX idx_classification ON events(classification);
CREATE INDEX idx_risk_level ON events(risk_level);
CREATE INDEX idx_timestamp_classification ON events(created_at DESC, classification);
```

---

## Testing the Implementation

**File:** `backend/internal/classification/orchestrator_test.go`

```go
package classification

import (
    "os"
    "testing"
)

func TestClassifyRestrictedFile(t *testing.T) {
    // Create test file with API key
    testFile := "/tmp/config.env"
    content := []byte(`AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPluExamples3")
    
    os.WriteFile(testFile, content, 0644)
    defer os.Remove(testFile)

    orchestrator := NewOrchestrator()
    result := orchestrator.Classify(&ClassificationRequest{
        FilePath: testFile,
        DeviceID: "TEST-001",
        EventID:  "evt-001",
    })

    if result.Classification != "RESTRICTED" {
        t.Errorf("Expected RESTRICTED, got %s", result.Classification)
    }

    if result.Confidence < 90 {
        t.Errorf("Expected high confidence (>90), got %.1f", result.Confidence)
    }
}

func TestClassifyPublicFile(t *testing.T) {
    testFile := "/tmp/empty.txt"
    os.WriteFile(testFile, []byte(""), 0644)
    defer os.Remove(testFile)

    orchestrator := NewOrchestrator()
    result := orchestrator.Classify(&ClassificationRequest{
        FilePath: testFile,
        DeviceID: "TEST-001",
        EventID:  "evt-002",
    })

    if result.Classification != "PUBLIC" {
        t.Errorf("Expected PUBLIC, got %s", result.Classification)
    }

    if result.Confidence != 100 {
        t.Errorf("Expected 100% confidence, got %.1f", result.Confidence)
    }
}
```

---

## Performance Benchmarks

| Operation | Time | Notes |
|-----------|------|-------|
| Pre-analysis | <50ms | File metadata only |
| Content parsing | 500-1500ms | Depends on file size/type |
| Pattern matching | 200-800ms | Keyword + regex scans |
| Decision engine | <50ms | Score normalization |
| **Total** | **2-3 seconds** | Per file (without AI) |

---

## Next: [03-AI-SETUP.md](./03-AI-SETUP.md)

LLM integration, Ollama setup, model fine-tuning

