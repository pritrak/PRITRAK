# 🔀 PRITRAK AI Classification: DLP Integration & Deployment

**Version:** 1.0  
**Date:** January 11, 2026  
**Complexity:** Advanced  
**Timeline:** 2-3 weeks for full integration  

---

## Overview: How It All Works Together

```
┌─────────────────────────────────────────────────────────────┐
│               COMPLETE CLASSIFICATION FLOW              │
├─────────────────────────────────────────────────────────────┤
│                                                        │
│  1. FILE EVENT (Agent)                               │
│     File created/deleted/modified                    │
│                 │                                    │
│                 ▼                                    │
│  2. TRANSMISSION (WebSocket)                          │
│     Event sent to backend                             │
│                 │                                    │
│                 ▼                                    │
│  3. CLASSIFICATION (Backend)                          │
│     ┌────────────────────────────────────────────────────┐   │
│     │  a) Pre-analysis (filename/ext)              │   │
│     │  b) Content parsing                          │   │
│     │  c) Pattern matching                         │   │
│     │  d) AI enhancement (Ollama)                │   │
│     │  e) Final decision (score + confidence)     │   │
│     └────────────────────────────────────────────────────┘   │
│                 │                                    │
│                 ▼                                    │
│  4. ENRICHMENT (Database)                            │
│     Event + classification → PostgreSQL/SQLite      │
│                 │                                    │
│                 ▼                                    │
│  5. POLICY CHECK (Rules Engine)                      │
│     ┌────────────────────────────────────────────────────┐   │
│     │  RESTRICTED + USB? → BLOCK                │   │
│     │  CONFIDENTIAL + Email? → WARN              │   │
│     │  HIGH Risk? → ALERT SOC                  │   │
│     └────────────────────────────────────────────────────┘   │
│                 │                                    │
│                 ▼                                    │
│  6. FRONTEND SYNC (WebSocket)                         │
│     Dashboard updated in real-time                    │
│                 │                                    │
│                 ▼                                    │
│  7. USER SEES (Dashboard)                             │
│     Event log with classification + action            │
│                                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1: Event Handler Integration

**File:** `backend/internal/endpoints/events.go`

```go
package endpoints

import (
    "context"
    "encoding/json"
    "log"
    "time"
    
    "your-app/internal/classification"
    "your-app/internal/classification/ai"
    "your-app/internal/db"
    "your-app/internal/models"
)

// EventHandler manages incoming file events
type EventHandler struct {
    db                 db.Database
    classifier         *classification.Orchestrator
    aiEnhancer         *ai.AIEnhancer
    policyEngine       *PolicyEngine
    wsHub             *WebSocketHub
    cache              *classification.ClassificationCache
}

// NewEventHandler creates event handler
func NewEventHandler(
    database db.Database,
    wsHub *WebSocketHub,
) *EventHandler {
    return &EventHandler{
        db:         database,
        classifier: classification.NewOrchestrator(),
        aiEnhancer: ai.NewAIEnhancer(
            ai.NewOllamaClient("http://localhost:11434", "mistral"),
        ),
        policyEngine: NewPolicyEngine(),
        wsHub:        wsHub,
        cache:        classification.NewClassificationCache(24 * time.Hour),
    }
}

// HandleFileEvent processes incoming file event
func (eh *EventHandler) HandleFileEvent(ctx context.Context, event *models.FileEvent) error {
    log.Printf("[EVENT] Processing: %s (type: %s)", event.FilePath, event.EventType)

    startTime := time.Now()

    // 1. Classify file
    classResult := eh.classifier.Classify(&classification.ClassificationRequest{
        FilePath: event.FilePath,
        DeviceID: event.DeviceID,
        EventID:  event.EventID,
    })

    // 2. Enhance with AI if confidence < 85%
    if classResult.Confidence < 85 {
        aiResult, err := eh.aiEnhancer.Enhance(&classification.ScoringResult{
            Language: "en", // Detected during classification
            Scores: map[string]float32{
                "PUBLIC":       0,
                "INTERNAL":     0,
                "CONFIDENTIAL": 0,
                "RESTRICTED":   0,
            },
        })

        if err == nil && aiResult.Confidence > classResult.Confidence {
            classResult.Classification = aiResult.Classification
            classResult.Confidence = aiResult.Confidence
            classResult.Justification = aiResult.Reasoning
        }
    }

    // 3. Enrich event with classification
    enrichedEvent := &models.EnrichedFileEvent{
        FileEvent:       event,
        Classification:  classResult.Classification,
        Confidence:      classResult.Confidence,
        RiskLevel:       classResult.RiskLevel,
        Justification:   classResult.Justification,
        ProcessingTimeMs: classResult.ProcessingMs,
        EnrichedAt:      time.Now(),
    }

    // 4. Save to database
    if err := eh.db.SaveEnrichedEvent(ctx, enrichedEvent); err != nil {
        log.Printf("[ERROR] Failed to save event: %v", err)
        return err
    }

    // 5. Apply policy-based actions
    action, actionReason := eh.policyEngine.EvaluatePolicy(enrichedEvent)
    enrichedEvent.Action = action
    enrichedEvent.ActionReason = actionReason

    // 6. Update with action
    if err := eh.db.UpdateEventAction(ctx, event.EventID, action, actionReason); err != nil {
        log.Printf("[ERROR] Failed to update action: %v", err)
    }

    // 7. Broadcast to connected clients
    eh.wsHub.Broadcast(&WebSocketMessage{
        Type:    "event_classified",
        Payload: enrichedEvent,
        Timestamp: time.Now(),
    })

    log.Printf("[CLASSIFIED] %s -> %s (%.1f%% confidence, %dms)",
        event.FilePath,
        classResult.Classification,
        classResult.Confidence,
        classResult.ProcessingMs,
    )

    return nil
}
```

---

## Step 2: Policy Engine

**File:** `backend/internal/policy/engine.go`

```go
package policy

import (
    "fmt"
    "your-app/internal/models"
)

// PolicyRule defines a classification-based action
type PolicyRule struct {
    Classification string
    EventType      string // CREATE, DELETE, MODIFY, TRANSFER, USB_ACCESS, EMAIL_SEND, etc.
    Action         string // ALLOW, WARN, BLOCK, ESCALATE
    Priority       int    // 1-10, higher = execute first
}

// PolicyEngine applies classification rules
type PolicyEngine struct {
    rules []PolicyRule
}

// NewPolicyEngine creates engine with default rules
func NewPolicyEngine() *PolicyEngine {
    return &PolicyEngine{
        rules: []PolicyRule{
            // RESTRICTED data policies (highest priority)
            {Classification: "RESTRICTED", EventType: "TRANSFER", Action: "BLOCK", Priority: 10},
            {Classification: "RESTRICTED", EventType: "USB_ACCESS", Action: "BLOCK", Priority: 10},
            {Classification: "RESTRICTED", EventType: "EMAIL_SEND", Action: "BLOCK", Priority: 10},
            {Classification: "RESTRICTED", EventType: "CLOUD_UPLOAD", Action: "BLOCK", Priority: 10},

            // CONFIDENTIAL data policies
            {Classification: "CONFIDENTIAL", EventType: "USB_ACCESS", Action: "WARN", Priority: 8},
            {Classification: "CONFIDENTIAL", EventType: "EMAIL_SEND", Action: "WARN", Priority: 8},
            {Classification: "CONFIDENTIAL", EventType: "TRANSFER", Action: "WARN", Priority: 8},

            // HIGH risk confidence triggers escalation
            {Classification: "INTERNAL", EventType: "MODIFY", Action: "ALLOW", Priority: 5},
            {Classification: "PUBLIC", EventType: "", Action: "ALLOW", Priority: 1},
        },
    }
}

// EvaluatePolicy returns action for event
func (pe *PolicyEngine) EvaluatePolicy(event *models.EnrichedFileEvent) (action string, reason string) {
    // Find matching rule (highest priority first)
    var bestRule *PolicyRule
    bestPriority := 0

    for i, rule := range pe.rules {
        if rule.Priority <= bestPriority {
            continue
        }

        if rule.Classification != event.Classification {
            continue
        }

        if rule.EventType != "" && rule.EventType != event.EventType {
            continue
        }

        bestRule = &pe.rules[i]
        bestPriority = rule.Priority
    }

    if bestRule == nil {
        return "ALLOW", "No matching policy rule"
    }

    reason = fmt.Sprintf("Policy: %s + %s -> %s",
        event.Classification,
        event.EventType,
        bestRule.Action,
    )

    return bestRule.Action, reason
}

// AddRule adds custom policy rule
func (pe *PolicyEngine) AddRule(rule PolicyRule) {
    pe.rules = append(pe.rules, rule)
}
```

---

## Step 3: Database Schema

```sql
-- Create enriched events table
CREATE TABLE IF NOT EXISTS enriched_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id VARCHAR(255) UNIQUE NOT NULL,
    device_id VARCHAR(255) NOT NULL,
    file_path TEXT NOT NULL,
    file_size BIGINT,
    file_extension VARCHAR(20),
    event_type VARCHAR(50) NOT NULL,
    username VARCHAR(255),
    
    -- Classification results
    classification VARCHAR(50) NOT NULL,
    confidence_score DECIMAL(5,2),
    risk_level VARCHAR(20),
    justification TEXT,
    
    -- Detection details
    detected_keywords TEXT,  -- JSON array
    pattern_matches TEXT,    -- JSON array
    language VARCHAR(5),
    processing_time_ms INTEGER,
    
    -- Policy action
    action VARCHAR(50),
    action_reason TEXT,
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    enriched_at TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes for performance
CREATE INDEX idx_device_id ON enriched_events(device_id);
CREATE INDEX idx_classification ON enriched_events(classification);
CREATE INDEX idx_risk_level ON enriched_events(risk_level);
CREATE INDEX idx_created_at ON enriched_events(created_at DESC);
CREATE INDEX idx_composite ON enriched_events(classification, risk_level, created_at DESC);

-- Create incidents table for high-risk events
CREATE TABLE IF NOT EXISTS incidents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id VARCHAR(255) UNIQUE NOT NULL,
    classification VARCHAR(50),
    risk_level VARCHAR(20),
    device_id VARCHAR(255),
    user_id VARCHAR(255),
    status VARCHAR(50) DEFAULT 'OPEN',  -- OPEN, INVESTIGATING, RESOLVED
    severity VARCHAR(50),  -- LOW, MEDIUM, HIGH, CRITICAL
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP,
    FOREIGN KEY (event_id) REFERENCES enriched_events(event_id)
);
```

---

## Step 4: Frontend Integration

**File:** `frontend/src/components/EventLog.vue` (Vue 3)

```vue
<template>
  <div class="event-log">
    <div class="filters">
      <select v-model="filters.classification">
        <option value="">All Classifications</option>
        <option value="PUBLIC">PUBLIC</option>
        <option value="INTERNAL">INTERNAL</option>
        <option value="CONFIDENTIAL">CONFIDENTIAL</option>
        <option value="RESTRICTED">RESTRICTED</option>
      </select>

      <select v-model="filters.riskLevel">
        <option value="">All Risk Levels</option>
        <option value="LOW">LOW</option>
        <option value="MEDIUM">MEDIUM</option>
        <option value="HIGH">HIGH</option>
        <option value="CRITICAL">CRITICAL</option>
      </select>

      <input 
        type="text" 
        v-model="searchQuery" 
        placeholder="Search events..."
      />
    </div>

    <table class="events-table">
      <thead>
        <tr>
          <th>Timestamp</th>
          <th>File Path</th>
          <th>Classification</th>
          <th>Confidence</th>
          <th>Risk Level</th>
          <th>Action</th>
          <th>Details</th>
        </tr>
      </thead>
      <tbody>
        <tr 
          v-for="event in filteredEvents" 
          :key="event.id"
          :class="`risk-${event.riskLevel.toLowerCase()}`"
        >
          <td>{{ formatTime(event.createdAt) }}</td>
          <td>{{ truncate(event.filePath, 50) }}</td>
          <td>
            <span :class="`badge-${event.classification.toLowerCase()}`">
              {{ event.classification }}
            </span>
          </td>
          <td>
            <div class="confidence-bar">
              <div 
                class="confidence-fill" 
                :style="{ width: event.confidence + '%' }"
              ></div>
              <span>{{ event.confidence.toFixed(1) }}%</span>
            </div>
          </td>
          <td>
            <span :class="`tag-${event.riskLevel.toLowerCase()}`">
              {{ event.riskLevel }}
            </span>
          </td>
          <td>
            <span :class="`action-${event.action.toLowerCase()}`">
              {{ event.action }}
            </span>
          </td>
          <td>
            <button @click="showDetails(event)">View</button>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- Detail modal -->
    <div v-if="selectedEvent" class="modal">
      <div class="modal-content">
        <h3>Event Details</h3>
        <div class="detail-grid">
          <div><strong>File Path:</strong> {{ selectedEvent.filePath }}</div>
          <div><strong>Classification:</strong> {{ selectedEvent.classification }}</div>
          <div><strong>Confidence:</strong> {{ selectedEvent.confidence }}%</div>
          <div><strong>Risk Level:</strong> {{ selectedEvent.riskLevel }}</div>
          <div><strong>Processing Time:</strong> {{ selectedEvent.processingTimeMs }}ms</div>
          <div><strong>Justification:</strong> {{ selectedEvent.justification }}</div>
          <div v-if="selectedEvent.detectedKeywords">
            <strong>Keywords:</strong> {{ selectedEvent.detectedKeywords.join(", ") }}
          </div>
          <div v-if="selectedEvent.patternMatches">
            <strong>Patterns:</strong> {{ selectedEvent.patternMatches.join(", ") }}
          </div>
        </div>
        <button @click="selectedEvent = null">Close</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const events = ref([])
const filters = ref({
  classification: '',
  riskLevel: ''
})
const searchQuery = ref('')
const selectedEvent = ref(null)
const ws = ref(null)

const filteredEvents = computed(() => {
  return events.value.filter(event => {
    if (filters.value.classification && event.classification !== filters.value.classification) return false
    if (filters.value.riskLevel && event.riskLevel !== filters.value.riskLevel) return false
    if (searchQuery.value && !event.filePath.includes(searchQuery.value)) return false
    return true
  })
})

onMounted(() => {
  // Connect to WebSocket
  ws.value = new WebSocket('ws://localhost:8080/ws')
  
  ws.value.onmessage = (e) => {
    const message = JSON.parse(e.data)
    if (message.type === 'event_classified') {
      events.value.unshift(message.payload)
      if (events.value.length > 1000) {
        events.value.pop() // Keep only last 1000
      }
    }
  }
})

onUnmounted(() => {
  if (ws.value) ws.value.close()
})

const formatTime = (timestamp) => {
  return new Date(timestamp).toLocaleString()
}

const truncate = (str, len) => {
  return str.length > len ? str.substring(0, len) + '...' : str
}

const showDetails = (event) => {
  selectedEvent.value = event
}
</script>

<style scoped>
.event-log {
  padding: 20px;
}

.filters {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.events-table {
  width: 100%;
  border-collapse: collapse;
}

.events-table th {
  background-color: #f5f5f5;
  padding: 12px;
  text-align: left;
  font-weight: 600;
}

.events-table td {
  padding: 12px;
  border-bottom: 1px solid #e0e0e0;
}

.risk-critical { background-color: #ffe0e0; }
.risk-high { background-color: #ffe8cc; }
.risk-medium { background-color: #fff8cc; }
.risk-low { background-color: #f0f0f0; }

.badge-public { background-color: #d4edda; color: #155724; padding: 4px 8px; border-radius: 4px; }
.badge-internal { background-color: #d1ecf1; color: #0c5460; padding: 4px 8px; border-radius: 4px; }
.badge-confidential { background-color: #fff3cd; color: #856404; padding: 4px 8px; border-radius: 4px; }
.badge-restricted { background-color: #f8d7da; color: #721c24; padding: 4px 8px; border-radius: 4px; }

.confidence-bar {
  display: flex;
  align-items: center;
  gap: 8px;
}

.confidence-fill {
  background-color: #4caf50;
  height: 20px;
  border-radius: 3px;
  min-width: 30px;
}

.tag-low { background-color: #d4edda; color: #155724; }
.tag-medium { background-color: #fff3cd; color: #856404; }
.tag-high { background-color: #f8d7da; color: #721c24; }
.tag-critical { background-color: #721c24; color: white; }

.action-allow { color: #28a745; }
.action-warn { color: #ff9800; }
.action-block { color: #f44336; }
.action-escalate { color: #9c27b0; }
</style>
```

---

## Step 5: Performance Testing

**File:** `backend/tests/classification_benchmark_test.go`

```go
package tests

import (
    "fmt"
    "os"
    "testing"
    "time"
    
    "your-app/internal/classification"
)

func BenchmarkClassification(b *testing.B) {
    // Create test files
    testFiles := createTestFiles(10)
    defer cleanupTestFiles(testFiles)

    orchestrator := classification.NewOrchestrator()

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        for _, file := range testFiles {
            orchestrator.Classify(&classification.ClassificationRequest{
                FilePath: file,
                DeviceID: "test-device",
                EventID:  fmt.Sprintf("evt-%d", i),
            })
        }
    }
}

func TestThroughput(t *testing.T) {
    orchestrator := classification.NewOrchestrator()
    testFile := createTestFile("confidential.xlsx", 100000)
    defer os.Remove(testFile)

    // Classify 100 times and measure throughput
    start := time.Now()
    for i := 0; i < 100; i++ {
        orchestrator.Classify(&classification.ClassificationRequest{
            FilePath: testFile,
            DeviceID: "test-device",
            EventID:  fmt.Sprintf("evt-%d", i),
        })
    }
    elapsed := time.Since(start)

    throughput := float64(100) / elapsed.Seconds()
    avgTime := elapsed.Milliseconds() / 100

    t.Logf("Throughput: %.1f files/sec", throughput)
    t.Logf("Avg time per file: %dms", avgTime)

    if avgTime > 5000 {
        t.Errorf("Classification too slow: %dms (target: <5000ms)", avgTime)
    }
}
```

---

## Step 6: Deployment Checklist

### Pre-Deployment
- [ ] All tests passing (unit + integration + load)
- [ ] Code reviewed and approved
- [ ] Security audit completed
- [ ] Database schema verified
- [ ] Ollama service configured and tested
- [ ] Monitoring/alerting configured
- [ ] Backup strategy in place

### Deployment Steps

```bash
#!/bin/bash
# deploy.sh

# 1. Build backend with classification module
echo "Building backend..."
cd backend
go build -o dap-server ./cmd/server/main.go

# 2. Deploy database migrations
echo "Running database migrations..."
go run ./cmd/migrate/main.go up

# 3. Start Ollama service
echo "Starting Ollama..."
sudo systemctl start ollama

# 4. Verify Ollama
curl http://localhost:11434/api/tags

# 5. Start backend service
echo "Starting DLP service..."
sudo systemctl restart dap-backend

# 6. Run health checks
echo "Running health checks..."
sleep 5
curl http://localhost:8080/health

# 7. Verify classification working
echo "Testing classification..."
curl -X POST http://localhost:8080/api/classify \
  -H "Content-Type: application/json" \
  -d '{"file_path": "/tmp/test.txt"}'

echo "Deployment complete!"
```

### Post-Deployment
- [ ] Monitor logs for errors
- [ ] Track classification metrics
- [ ] Verify frontend updates in real-time
- [ ] Test policy enforcement
- [ ] Collect performance baselines

---

## Monitoring & Observability

### Metrics to Track

```go
// Add metrics collection
type ClassificationMetrics struct {
    TotalClassified      int64
    AverageTimeMs        float64
    ByClassification     map[string]int64
    ByRiskLevel          map[string]int64
    PolicyBlockCount     int64
    PolicyWarnCount      int64
    AIEnhancementCount   int64
}

func (em *EventHandler) recordMetrics(result *ClassificationResult) {
    em.metrics.TotalClassified++
    em.metrics.AverageTimeMs = (em.metrics.AverageTimeMs + float64(result.ProcessingMs)) / 2
    em.metrics.ByClassification[result.Classification]++
    em.metrics.ByRiskLevel[result.RiskLevel]++
    
    // Log for monitoring
    log.Printf("METRIC: classification=%s risk=%s time=%dms confidence=%.1f",
        result.Classification,
        result.RiskLevel,
        result.ProcessingMs,
        result.Confidence,
    )
}
```

### Alert Rules

```yaml
alertrules:
  - name: HighBlockRate
    condition: rate(policy_blocks[5m]) > 10
    action: notify_soc
  
  - name: ClassificationLatency
    condition: classification_time_p99 > 5000  # 5 seconds
    action: page_oncall
  
  - name: OllamaDown
    condition: ollama_health != 1
    action: fallback_to_rules_only
  
  - name: RestrictedDataDetected
    condition: classification == 'RESTRICTED'
    action: escalate_incident
```

---

## Rollback Plan

If issues arise:

```bash
# 1. Disable classification (keep pre-analysis only)
sqlite3 config.db "UPDATE classification_config SET enabled = false;"

# 2. Revert to previous backend version
git revert HEAD
go build -o dap-server ./cmd/server/main.go
sudo systemctl restart dap-backend

# 3. Clear classification cache
Redis: FLUSHDB

# 4. Verify
curl http://localhost:8080/health
```

---

## Next Steps

1. **[Implementation](./02-IMPLEMENTATION.md)** - Start coding
2. **[Testing](./05-TESTING.md)** - Comprehensive test strategy
3. **[Operations](./06-OPERATIONS.md)** - Ongoing maintenance

---

**For questions or issues, open an issue on GitHub:**  
https://github.com/pritrak/PRITRAK/issues

