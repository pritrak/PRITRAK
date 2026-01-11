# 🤖 PRITRAK AI Classification: LLM Setup & Configuration

**Version:** 1.0  
**Date:** January 11, 2026  
**Focus:** Local LLM deployment with Ollama  
**AI Model:** Mistral 7B (recommended) / Llama 2 7B  

---

## Why Local LLM?

| Concern | Cloud API | Local LLM |
|---------|-----------|----------|
| **Data Privacy** | ❌ Data sent externally | ✅ On-premises only |
| **Cost** | $$$$ Per API call | $ One-time setup |
| **Latency** | 2-5 seconds | <1 second (GPU) |
| **Compliance** | GDPR/HIPAA risks | ✅ Full control |
| **No Dependency** | External service required | ✅ Self-contained |

---

## Hardware Requirements

### Minimum (CPU Inference)
- 16GB RAM
- 50GB SSD space
- Processing time: 10-20 seconds per file

### Recommended (GPU Acceleration) ⚡
- 12GB+ VRAM GPU (NVIDIA RTX 3080 or better)
- 32GB RAM
- 100GB SSD space
- Processing time: 1-3 seconds per file

### Ideal (Production)
- 24GB+ VRAM GPU (NVIDIA A100 or RTX 4090)
- 64GB RAM
- NVMe SSD with 200GB+
- Processing time: 500-800ms per file

---

## Step 1: Install Ollama

### Linux/macOS

```bash
# Download and install
curl -fsSL https://ollama.ai/install.sh | sh

# Start Ollama service
ollama serve

# In another terminal, verify
ollama --version
```

### Windows

```bash
# Download installer from https://ollama.ai
# Run: OllamaSetup.exe

# Verify installation
ollama --version

# Start service
ollama serve
```

### Docker (Recommended for Enterprise)

```bash
# Pull Ollama image
docker pull ollama/ollama

# Run with GPU support
docker run --gpus all -d \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama \
  ollama/ollama

# Test
curl http://localhost:11434/api/tags
```

---

## Step 2: Download and Configure Models

### Option A: Mistral 7B (Recommended)

```bash
# Download Mistral (4GB)
ollama pull mistral

# Verify
ollama list
```

**Characteristics:**
- ✅ Excellent accuracy for classification
- ✅ Fast inference (1-2 seconds)
- ✅ Smaller memory footprint
- ✅ Better English/French support

### Option B: Llama 2 7B

```bash
# Download Llama 2 (4GB)
ollama pull llama2
```

**Characteristics:**
- ✅ Strong overall performance
- ✅ Large community/documentation
- ❌ Slightly slower than Mistral
- ✅ Good multilingual support

### Option C: Neural Chat (Fastest)

```bash
# Download Neural Chat (3.8GB)
ollama pull neural-chat
```

**Characteristics:**
- ✅ Fastest inference (500ms)
- ✅ Optimized for conversations
- ❌ Slightly lower accuracy
- ✅ Lightweight

---

## Step 3: Test Ollama Connection

**File:** `backend/internal/classification/ai/ollama_client.go`

```go
package ai

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

// OllamaClient handles communication with Ollama
type OllamaClient struct {
    baseURL    string
    model      string
    timeout    time.Duration
    httpClient *http.Client
}

// GenerateRequest is sent to Ollama
type GenerateRequest struct {
    Model  string `json:"model"`
    Prompt string `json:"prompt"`
    Stream bool   `json:"stream"`
    Options Options `json:"options,omitempty"`
}

// Options for Ollama generation
type Options struct {
    Temperature float32 `json:"temperature,omitempty"`
    TopP       float32 `json:"top_p,omitempty"`
    TopK       int     `json:"top_k,omitempty"`
}

// GenerateResponse from Ollama
type GenerateResponse struct {
    Model              string    `json:"model"`
    CreatedAt          time.Time `json:"created_at"`
    Response           string    `json:"response"`
    Done               bool      `json:"done"`
    Context            []int     `json:"context,omitempty"`
    TotalDuration      int64     `json:"total_duration,omitempty"`
    LoadDuration       int64     `json:"load_duration,omitempty"`
    PromptEvalCount    int       `json:"prompt_eval_count,omitempty"`
    PromptEvalDuration int64     `json:"prompt_eval_duration,omitempty"`
    EvalCount          int       `json:"eval_count,omitempty"`
    EvalDuration       int64     `json:"eval_duration,omitempty"`
}

// NewOllamaClient creates a new Ollama client
func NewOllamaClient(baseURL, model string) *OllamaClient {
    return &OllamaClient{
        baseURL: baseURL,
        model:   model,
        timeout: 30 * time.Second,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

// Health checks if Ollama is running
func (oc *OllamaClient) Health() error {
    resp, err := oc.httpClient.Get(fmt.Sprintf("%s/api/tags", oc.baseURL))
    if err != nil {
        return fmt.Errorf("ollama unreachable: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("ollama health check failed: status %d", resp.StatusCode)
    }

    return nil
}

// Generate sends prompt to Ollama and gets response
func (oc *OllamaClient) Generate(prompt string) (string, error) {
    reqBody := GenerateRequest{
        Model:  oc.model,
        Prompt: prompt,
        Stream: false,
        Options: Options{
            Temperature: 0.2,  // Lower = more deterministic
            TopP:        0.9,
            TopK:        40,
        },
    }

    jsonBody, err := json.Marshal(reqBody)
    if err != nil {
        return "", err
    }

    resp, err := oc.httpClient.Post(
        fmt.Sprintf("%s/api/generate", oc.baseURL),
        "application/json",
        bytes.NewBuffer(jsonBody),
    )
    if err != nil {
        return "", fmt.Errorf("ollama request failed: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(resp.Body)
        return "", fmt.Errorf("ollama error (status %d): %s", resp.StatusCode, string(body))
    }

    var genResp GenerateResponse
    if err := json.NewDecoder(resp.Body).Decode(&genResp); err != nil {
        return "", fmt.Errorf("failed to parse ollama response: %w", err)
    }

    return genResp.Response, nil
}

// ListModels returns available models in Ollama
func (oc *OllamaClient) ListModels() ([]string, error) {
    resp, err := oc.httpClient.Get(fmt.Sprintf("%s/api/tags", oc.baseURL))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }

    var models []string
    if modelsData, ok := result["models"].([]interface{}); ok {
        for _, m := range modelsData {
            if model, ok := m.(map[string]interface{}); ok {
                if name, ok := model["name"].(string); ok {
                    models = append(models, name)
                }
            }
        }
    }

    return models, nil
}
```

**Test Script:**

```bash
#!/bin/bash
# test_ollama.sh

echo "1. Testing Ollama connection..."
curl -s http://localhost:11434/api/tags | jq '.'

echo ""
echo "2. Testing Mistral model..."
curl -X POST http://localhost:11434/api/generate -H "Content-Type: application/json" -d '{
  "model": "mistral",
  "prompt": "Classify this document: Q1 Financial Forecast. Categories: PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED.",
  "stream": false
}' | jq '.response'

echo ""
echo "3. Checking inference speed..."
time curl -X POST http://localhost:11434/api/generate -H "Content-Type: application/json" -d '{
  "model": "mistral",
  "prompt": "Hello",
  "stream": false
}' > /dev/null
```

---

## Step 4: Optimize Model for Classification

### Create Fine-Tuned Prompt Template

**File:** `backend/internal/classification/ai/prompt_templates.go`

```go
package ai

import (
    "fmt"
)

// ClassificationPromptTemplate builds optimized prompt for classification
type ClassificationPromptTemplate struct {
    IncludeContext bool
    Language       string // "en" or "fr"
}

// BuildPrompt creates the LLM prompt
func (cpt *ClassificationPromptTemplate) BuildPrompt(
    filename string,
    content string,
    keywords []string,
    patterns []string,
) string {
    if cpt.Language == "fr" {
        return cpt.buildFrenchPrompt(filename, content, keywords, patterns)
    }
    return cpt.buildEnglishPrompt(filename, content, keywords, patterns)
}

func (cpt *ClassificationPromptTemplate) buildEnglishPrompt(
    filename string,
    content string,
    keywords []string,
    patterns []string,
) string {
    prompt := `You are a data classification expert. Your job is to classify documents according to NIST security standards.

DOCUMENT TO CLASSIFY:
Filename: ` + filename + `
Content (first 2000 chars): ` + content[:min(len(content), 2000)] + `

CLASSIFICATION CATEGORIES:
1. PUBLIC - No security risk, can be shared externally
2. INTERNAL - For internal use only, limited disruption if exposed
3. CONFIDENTIAL - Restricted access, significant business impact if exposed
4. RESTRICTED - Trade secrets, credentials, severe consequences if exposed

DETECTED INDICATORS:
`

    if len(keywords) > 0 {
        prompt += fmt.Sprintf("Keywords found: %v\n", keywords)
    }
    if len(patterns) > 0 {
        prompt += fmt.Sprintf("Sensitive patterns detected: %v\n", patterns)
    }

    prompt += `
REQUIRED RESPONSE FORMAT (JSON):
{
  "classification": "CATEGORY_NAME",
  "confidence": 0-100,
  "key_factors": ["factor1", "factor2"],
  "reasoning": "brief explanation"
}

Respond ONLY with the JSON object, no other text.`

    return prompt
}

func (cpt *ClassificationPromptTemplate) buildFrenchPrompt(
    filename string,
    content string,
    keywords []string,
    patterns []string,
) string {
    // Similar but in French
    prompt := `Vous êtes un expert en classification des données. Classifiez ce document selon les normes NIST.

DOCUMENT À CLASSIFIER:
Nom: ` + filename + `
Contenu: ` + content[:min(len(content), 2000)] + `

CATAGORIES:
1. PUBLIC - Aucun risque
2. INTERNE - Usage interne seulement
3. CONFIDENTIEL - Accès restreint
4. RESTREINIT - Secrets commerciaux, résultats graves si exposé

RÉPONSE EN JSON:
{
  "classification": "CATEGORY_NAME",
  "confidence": 0-100,
  "reasoning": "explication brève"
}
`
    return prompt
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

### Parse LLM Response

**File:** `backend/internal/classification/ai/response_parser.go`

```go
package ai

import (
    "encoding/json"
    "fmt"
    "regexp"
    "strconv"
    "strings"
)

// LLMClassificationResult from LLM
type LLMClassificationResult struct {
    Classification string   `json:"classification"`
    Confidence     int      `json:"confidence"`
    KeyFactors     []string `json:"key_factors,omitempty"`
    Reasoning      string   `json:"reasoning,omitempty"`
}

// ParseLLMResponse parses JSON response from Ollama
func ParseLLMResponse(response string) (*LLMClassificationResult, error) {
    // Extract JSON from response (may have extra text)
    jsonStr := extractJSON(response)
    if jsonStr == "" {
        return nil, fmt.Errorf("no JSON found in response")
    }

    var result LLMClassificationResult
    if err := json.Unmarshal([]byte(jsonStr), &result); err != nil {
        return nil, fmt.Errorf("failed to parse JSON: %w", err)
    }

    // Validate classification
    validClasses := map[string]bool{
        "PUBLIC":       true,
        "INTERNAL":     true,
        "CONFIDENTIAL": true,
        "RESTRICTED":   true,
    }

    if !validClasses[result.Classification] {
        return nil, fmt.Errorf("invalid classification: %s", result.Classification)
    }

    // Clamp confidence
    if result.Confidence < 0 {
        result.Confidence = 0
    }
    if result.Confidence > 100 {
        result.Confidence = 100
    }

    return &result, nil
}

// extractJSON extracts JSON object from text
func extractJSON(text string) string {
    // Find first { and last }
    start := strings.Index(text, "{")
    if start == -1 {
        return ""
    }

    end := strings.LastIndex(text, "}")
    if end == -1 || end <= start {
        return ""
    }

    return text[start : end+1]
}

// ValidateConfidence checks confidence reasonableness
func ValidateConfidence(conf int) bool {
    return conf >= 0 && conf <= 100
}

// MapLLMtoRiskLevel maps classification to risk level
func MapLLMtoRiskLevel(classification string) string {
    riskMap := map[string]string{
        "PUBLIC":       "LOW",
        "INTERNAL":     "LOW",
        "CONFIDENTIAL": "HIGH",
        "RESTRICTED":   "CRITICAL",
    }

    if risk, exists := riskMap[classification]; exists {
        return risk
    }
    return "MEDIUM"
}
```

---

## Step 5: Performance Optimization

### Caching Strategy

**File:** `backend/internal/classification/ai/cache.go`

```go
package ai

import (
    "crypto/sha256"
    "fmt"
    "sync"
    "time"
)

// ClassificationCache stores recent classification results
type ClassificationCache struct {
    cache map[string]*CachedResult
    mu    sync.RWMutex
    ttl   time.Duration
}

// CachedResult holds cached classification
type CachedResult struct {
    Classification string
    Confidence     float32
    RiskLevel      string
    Timestamp      time.Time
}

// NewClassificationCache creates cache with TTL
func NewClassificationCache(ttl time.Duration) *ClassificationCache {
    cache := &ClassificationCache{
        cache: make(map[string]*CachedResult),
        ttl:   ttl,
    }
    // Cleanup goroutine
    go cache.cleanup()
    return cache
}

// Get retrieves cached result if available
func (cc *ClassificationCache) Get(filePath string, fileHash string) *CachedResult {
    cc.mu.RLock()
    defer cc.mu.RUnlock()

    key := fmt.Sprintf("%s:%s", filePath, fileHash)
    if result, exists := cc.cache[key]; exists {
        if time.Since(result.Timestamp) < cc.ttl {
            return result
        }
    }
    return nil
}

// Set stores classification result
func (cc *ClassificationCache) Set(filePath string, fileHash string, result *CachedResult) {
    cc.mu.Lock()
    defer cc.mu.Unlock()

    key := fmt.Sprintf("%s:%s", filePath, fileHash)
    result.Timestamp = time.Now()
    cc.cache[key] = result
}

// ComputeFileHash computes SHA-256 hash of file content
func ComputeFileHash(content []byte) string {
    hash := sha256.Sum256(content)
    return fmt.Sprintf("%x", hash)
}

// cleanup removes expired entries
func (cc *ClassificationCache) cleanup() {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()

    for range ticker.C {
        cc.mu.Lock()
        now := time.Now()
        for key, result := range cc.cache {
            if now.Sub(result.Timestamp) > cc.ttl {
                delete(cc.cache, key)
            }
        }
        cc.mu.Unlock()
    }
}
```

---

## Step 6: Error Handling & Fallback

```go
// AIEnhancer with fallback
type AIEnhancer struct {
    ollamaClient      *OllamaClient
    cache             *ClassificationCache
    fallbackThreshold float32 // Use fallback if confidence < this
}

func (ae *AIEnhancer) Enhance(scoringResult *ScoringResult) (*EnhancementResult, error) {
    // Check if Ollama is available
    if err := ae.ollamaClient.Health(); err != nil {
        // Ollama unavailable, use fallback
        return ae.fallbackEnhance(scoringResult), nil
    }

    // Build prompt
    template := &ClassificationPromptTemplate{Language: scoringResult.Language}
    prompt := template.BuildPrompt(
        "", // filename
        "", // content
        scoringResult.DetectedKeywords,
        []string{}, // pattern matches
    )

    // Get LLM response
    response, err := ae.ollamaClient.Generate(prompt)
    if err != nil {
        // LLM call failed, use fallback
        return ae.fallbackEnhance(scoringResult), nil
    }

    // Parse response
    llmResult, err := ParseLLMResponse(response)
    if err != nil {
        // Parse failed, use fallback
        return ae.fallbackEnhance(scoringResult), nil
    }

    return &EnhancementResult{
        Classification: llmResult.Classification,
        Confidence:     float32(llmResult.Confidence),
        Reasoning:      llmResult.Reasoning,
    }, nil
}

func (ae *AIEnhancer) fallbackEnhance(scoringResult *ScoringResult) *EnhancementResult {
    // Use existing scoring without LLM
    bestClass := ""
    bestScore := float32(0)
    for class, score := range scoringResult.Scores {
        if score > bestScore {
            bestScore = score
            bestClass = class
        }
    }
    return &EnhancementResult{
        Classification: bestClass,
        Confidence:     min32(bestScore, 85), // Cap at 85% without AI
        Reasoning:      "LLM unavailable, using rule-based classification",
    }
}
```

---

## Performance Tuning

### Model Parameters

```yaml
# ollama_config.yaml
model_name: mistral
parameters:
  temperature: 0.2        # Lower = deterministic
  top_p: 0.9             # Nucleus sampling
  top_k: 40              # Consider top K tokens
  repeat_penalty: 1.1
  
performance:
  num_predict: 256       # Max tokens in response
  num_gpu: 1             # GPU layers
  num_thread: 8          # CPU threads
  batch_size: 4          # Batch for parallel requests
```

### Load Testing

```bash
# Generate 1000 classification requests in parallel
for i in {1..1000}; do
    (time curl -X POST http://localhost:11434/api/generate \
      -H "Content-Type: application/json" \
      -d '{"model":"mistral","prompt":"Classify: Financial report","stream":false}' \
      > /dev/null 2>&1) &
done

wait
echo "Load test complete"
```

---

## Monitoring & Logging

```go
// Log classification with AI timing
log.Printf("Classification: %s | Confidence: %.1f%% | AI Time: %dms | Total: %dms",
    decision.Classification,
    decision.Confidence,
    aiProcessingTime,
    totalProcessingTime,
)
```

---

## Next: [04-INTEGRATION.md](./04-INTEGRATION.md)

Full DLP sync, event pipeline, frontend updates

