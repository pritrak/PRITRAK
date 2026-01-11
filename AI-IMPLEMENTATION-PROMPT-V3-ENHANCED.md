# 🤖 PRITRAK V3.0 Real-Time Classification System - Complete AI Implementation Prompt

**CRITICAL: This prompt should be fed to GitHub Copilot, Claude, or your AI coding agent as a single continuous task. The AI should NOT STOP until 100% completion.**

---

## EXECUTIVE BRIEF - WHAT YOU'RE BUILDING

You are implementing a **real-time file classification engine** for PRITRAK Data Loss Prevention (DLP) system with **custom rule-based overrides**. Users create/modify files on Windows, your system classifies them instantly (<50ms), AND admins can create custom rules to reclassify files based on patterns.

**Current Problems:**
1. File created: `testt.txt` → Classification field shows EMPTY
2. No way for admins to customize classification rules
3. No UI widget to add/edit classification rules

**What You're Building (V3.0):**
1. ✅ Real-time file classification (Phase 0-4 engine)
2. ✅ Rule-based classification override system (NEW)
3. ✅ Classification rules management UI widget (NEW)
4. ✅ Real-time dashboard updates via WebSocket
5. ✅ Seamless empty→modified transitions with reclassification

---

## PART 1: ARCHITECTURE OVERVIEW (Updated)

### 1.1 Complete Data Flow

```
User Action: File Created/Modified on disk
    ↓
    ├─ [Windows IRP/USN Journal captures event]
    ├─ C++ DLP Agent receives event
    ├─ Run Classification Engine (Phases 0-4)
    │
    ├─ PHASE 0-4: STANDARD CLASSIFICATION (as before)
    │  Result: {classification: "PUBLIC", score: 65, confidence: 85}
    │
    └─ [NEW IN V3] PHASE 5: RULE ENGINE EVALUATION
       ├─ Load all admin-configured rules from database
       │
       ├─ For each rule (in priority order):
       │  ├─ Evaluate condition (field=keyword, value=match, operator=equals)
       │  ├─ If matches → Apply action (Classify As: PRIVATE/CONFIDENTIAL/RESTRICTED)
       │  ├─ Override Phase 0-4 score with rule classification
       │  └─ Track which rule triggered for audit
       │
       └─ Final Result: {classification: "PRIVATE", score: 100, ruleTriggered: "Rule-1", confidence: 100}
    ↓
Result: {classification, score, confidence, explanation, ruleTriggered}
    ↓
    ├─ [Update Database]
    ├─ [WebSocket Push to Dashboard]
    └─ [Audit Log]
    ↓
Dashboard updates instantly (user sees classification within 50ms)
```

### 1.2 Why This Architecture?

```
BEFORE (V1/V2):
   Fixed scoring rules → Works for 90% of files → Breaks for custom requirements

AFTER (V3):
   Phase 0-4 scoring → Works for 90% of files
   + Admin Rules Engine → Handles custom 10% → Flexible + Enterprise-ready
   = Perfect blend of automation + control
```

---

## PART 2: CLASSIFICATION RULES SYSTEM (NEW FOR V3)

### 2.1 Database Schema for Rules

```sql
-- FILE: database/schema.sql (ADDITIONS)

CREATE TABLE IF NOT EXISTS classification_rules (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    enabled BOOLEAN DEFAULT 1,
    priority INTEGER DEFAULT 100,  -- Lower = higher priority (0-1000)
    
    -- Condition (what triggers the rule)
    condition_field TEXT NOT NULL,  -- "keyword", "file_extension", "file_size", "directory_path", "user_group", "content_pattern"
    condition_operator TEXT NOT NULL,  -- "equals", "contains", "matches_regex", "gt", "lt", "in_list"
    condition_value TEXT NOT NULL,  -- The actual value to match
    
    -- Action (what happens when rule triggers)
    action_type TEXT NOT NULL,  -- "classify_as", "quarantine", "notify", "require_approval"
    action_classification TEXT,  -- "PUBLIC", "PRIVATE", "CONFIDENTIAL", "RESTRICTED" (used if action_type="classify_as")
    
    -- Metadata
    created_by TEXT,  -- Admin username
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_system BOOLEAN DEFAULT 0  -- true if default rule, false if admin-created
);\n
CREATE INDEX idx_rules_priority ON classification_rules(priority, enabled);\n
CREATE INDEX idx_rules_enabled ON classification_rules(enabled);\n
CREATE TABLE IF NOT EXISTS rule_audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    rule_id INTEGER NOT NULL,
    file_path TEXT NOT NULL,
    triggered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    old_classification TEXT,
    new_classification TEXT,
    FOREIGN KEY(rule_id) REFERENCES classification_rules(id)
);
```

### 2.2 Rule Examples (What Admins Can Create)

```
RULE 1: "Payroll Files"
├─ Condition: keyword equals "payroll"
├─ Action: Classify as RESTRICTED
├─ Example: payroll_2024.xlsx → RESTRICTED

RULE 2: "Marketing Department"
├─ Condition: directory_path contains "C:\\Marketing"
├─ Action: Classify as CONFIDENTIAL
├─ Example: C:\\Marketing\\campaign.pptx → CONFIDENTIAL

RULE 3: "Internal Memos"
├─ Condition: content_pattern matches_regex "INTERNAL MEMO"
├─ Action: Classify as PRIVATE
├─ Example: memo.txt containing "INTERNAL MEMO" → PRIVATE

RULE 4: "Large Backups"
├─ Condition: file_size gt 100 MB
├─ Action: Classify as RESTRICTED (suspicious)
├─ Example: backup.zip 250MB → RESTRICTED

RULE 5: "Department Teams"
├─ Condition: user_group in_list ["Finance", "HR", "Legal"]
├─ Action: Classify as CONFIDENTIAL
├─ Example: Finance user creates file → CONFIDENTIAL

RULE 6: "Source Code"
├─ Condition: file_extension in_list [".cpp", ".py", ".js", ".java"]
├─ Action: Classify as INTERNAL
├─ Example: app.cpp → INTERNAL

RULE 7: "Public Templates"
├─ Condition: file_extension equals ".template"
├─ Action: Classify as PUBLIC
├─ Example: form.template → PUBLIC
```

### 2.3 Rule Engine Implementation (C++)

```cpp
// FILE: src/RuleEngine.cpp

#include <sqlite3.h>
#include <regex>
#include <vector>
#include <memory>

struct ClassificationRule {
    int id;
    std::string name;
    std::string description;
    bool enabled;
    int priority;
    
    // Condition
    std::string condition_field;  // "keyword", "file_extension", etc.
    std::string condition_operator;  // "equals", "contains", "matches_regex", etc.
    std::string condition_value;
    
    // Action
    std::string action_type;  // "classify_as"
    std::string action_classification;  // "PUBLIC", "PRIVATE", etc.
};

struct RuleEvaluationResult {
    bool matched;
    int rule_id;
    std::string rule_name;
    std::string new_classification;
    std::string explanation;
};

class RuleEngine {
private:
    sqlite3* db;
    std::vector<ClassificationRule> rules;
    std::mutex rulesMutex;
    
public:
    RuleEngine(const std::string& dbPath) {
        sqlite3_open(dbPath.c_str(), &db);
        load_rules_from_db();
    }
    
    ~RuleEngine() {
        sqlite3_close(db);
    }
    
    // Main evaluation function
    RuleEvaluationResult evaluate(const std::string& filePath, const ClassificationResult& phaseResult) {
        std::lock_guard<std::mutex> lock(rulesMutex);
        
        // Sort rules by priority (lower priority = checked first)
        auto sortedRules = rules;
        std::sort(sortedRules.begin(), sortedRules.end(),
                  [](const ClassificationRule& a, const ClassificationRule& b) {
                      return a.priority < b.priority;
                  });
        
        // Evaluate each rule in order
        for (const auto& rule : sortedRules) {
            if (!rule.enabled) continue;
            
            bool conditionMet = evaluate_condition(rule, filePath, phaseResult);
            
            if (conditionMet) {
                return RuleEvaluationResult{
                    true,
                    rule.id,
                    rule.name,
                    rule.action_classification,
                    "Rule '" + rule.name + "' triggered"
                };
            }
        }
        
        // No rule matched
        return RuleEvaluationResult{
            false,
            -1,
            "",
            phaseResult.classification,
            "No rules matched, using Phase 0-4 result"
        };
    }
    
private:
    bool evaluate_condition(const ClassificationRule& rule, 
                           const std::string& filePath,
                           const ClassificationResult& phaseResult) {
        
        // CONDITION FIELD: keyword
        if (rule.condition_field == "keyword") {
            std::string filename = get_filename(filePath);
            std::string lower_filename = to_lower(filename);
            std::string lower_value = to_lower(rule.condition_value);
            
            if (rule.condition_operator == "equals") {
                return lower_filename == lower_value;
            } else if (rule.condition_operator == "contains") {
                return lower_filename.find(lower_value) != std::string::npos;
            } else if (rule.condition_operator == "matches_regex") {
                try {
                    std::regex pattern(rule.condition_value, std::regex::icase);
                    return std::regex_search(lower_filename, pattern);
                } catch (...) {
                    return false;
                }
            }
        }
        
        // CONDITION FIELD: file_extension
        if (rule.condition_field == "file_extension") {
            std::string ext = get_extension(filePath);
            std::string lower_ext = to_lower(ext);
            std::string lower_value = to_lower(rule.condition_value);
            
            if (rule.condition_operator == "equals") {
                return lower_ext == lower_value || lower_ext == "." + lower_value;
            } else if (rule.condition_operator == "in_list") {
                // Parse comma-separated list: ".cpp, .py, .js"
                std::vector<std::string> extensions = split(rule.condition_value, ",");
                for (auto& e : extensions) {
                    std::string trimmed = trim(e);
                    if (to_lower(trimmed) == lower_ext || "." + to_lower(trimmed) == lower_ext) {
                        return true;
                    }
                }
                return false;
            }
        }
        
        // CONDITION FIELD: file_size
        if (rule.condition_field == "file_size") {
            size_t fileSize = get_file_size(filePath);
            size_t conditionSize = std::stoull(rule.condition_value) * 1024 * 1024;  // Convert MB to bytes
            
            if (rule.condition_operator == "gt") {
                return fileSize > conditionSize;
            } else if (rule.condition_operator == "lt") {
                return fileSize < conditionSize;
            } else if (rule.condition_operator == "equals") {
                return fileSize == conditionSize;
            }
        }
        
        // CONDITION FIELD: directory_path
        if (rule.condition_field == "directory_path") {
            std::string dir = get_directory(filePath);
            std::string lower_dir = to_lower(dir);
            std::string lower_value = to_lower(rule.condition_value);
            
            if (rule.condition_operator == "equals") {
                return lower_dir == lower_value;
            } else if (rule.condition_operator == "contains") {
                return lower_dir.find(lower_value) != std::string::npos;
            } else if (rule.condition_operator == "matches_regex") {
                try {
                    std::regex pattern(rule.condition_value, std::regex::icase);
                    return std::regex_search(lower_dir, pattern);
                } catch (...) {
                    return false;
                }
            }
        }
        
        // CONDITION FIELD: content_pattern
        if (rule.condition_field == "content_pattern") {
            std::string content = read_file_content(filePath, 1_MB);  // First 1MB
            std::string lower_content = to_lower(content);
            std::string lower_value = to_lower(rule.condition_value);
            
            if (rule.condition_operator == "contains") {
                return lower_content.find(lower_value) != std::string::npos;
            } else if (rule.condition_operator == "matches_regex") {
                try {
                    std::regex pattern(rule.condition_value, std::regex::icase);
                    return std::regex_search(content, pattern);
                } catch (...) {
                    return false;
                }
            }
        }
        
        // CONDITION FIELD: user_group (requires Windows API)
        if (rule.condition_field == "user_group") {
            std::string userGroup = get_file_owner_group(filePath);
            std::string lower_group = to_lower(userGroup);
            std::string lower_value = to_lower(rule.condition_value);
            
            if (rule.condition_operator == "equals") {
                return lower_group == lower_value;
            } else if (rule.condition_operator == "in_list") {
                std::vector<std::string> groups = split(rule.condition_value, ",");
                for (auto& g : groups) {
                    if (to_lower(trim(g)) == lower_group) {
                        return true;
                    }
                }
                return false;
            }
        }
        
        return false;
    }
    
    void load_rules_from_db() {
        std::lock_guard<std::mutex> lock(rulesMutex);
        rules.clear();
        
        const char* sql = "SELECT id, name, description, enabled, priority, condition_field, "
                         "condition_operator, condition_value, action_type, action_classification "
                         "FROM classification_rules WHERE enabled = 1 ORDER BY priority";
        
        sqlite3_stmt* stmt;
        if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) == SQLITE_OK) {
            while (sqlite3_step(stmt) == SQLITE_ROW) {
                ClassificationRule rule;
                rule.id = sqlite3_column_int(stmt, 0);
                rule.name = (const char*)sqlite3_column_text(stmt, 1);
                rule.description = (const char*)sqlite3_column_text(stmt, 2);
                rule.enabled = sqlite3_column_int(stmt, 3) != 0;
                rule.priority = sqlite3_column_int(stmt, 4);
                rule.condition_field = (const char*)sqlite3_column_text(stmt, 5);
                rule.condition_operator = (const char*)sqlite3_column_text(stmt, 6);
                rule.condition_value = (const char*)sqlite3_column_text(stmt, 7);
                rule.action_type = (const char*)sqlite3_column_text(stmt, 8);
                rule.action_classification = (const char*)sqlite3_column_text(stmt, 9);
                
                rules.push_back(rule);
            }
            sqlite3_finalize(stmt);
        }
    }
    
    void reload_rules() {
        load_rules_from_db();
    }
    
public:
    void add_rule(const ClassificationRule& rule) {
        const char* sql = "INSERT INTO classification_rules "
                         "(name, description, enabled, priority, condition_field, condition_operator, "
                         "condition_value, action_type, action_classification, created_by) "
                         "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)";
        
        sqlite3_stmt* stmt;
        if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) == SQLITE_OK) {
            sqlite3_bind_text(stmt, 1, rule.name.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 2, rule.description.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_int(stmt, 3, rule.enabled ? 1 : 0);
            sqlite3_bind_int(stmt, 4, rule.priority);
            sqlite3_bind_text(stmt, 5, rule.condition_field.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 6, rule.condition_operator.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 7, rule.condition_value.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 8, rule.action_type.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 9, rule.action_classification.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 10, "admin", -1, SQLITE_STATIC);
            
            if (sqlite3_step(stmt) == SQLITE_DONE) {
                reload_rules();  // Refresh rules after adding
            }
            sqlite3_finalize(stmt);
        }
    }
    
    void delete_rule(int ruleId) {
        const char* sql = "DELETE FROM classification_rules WHERE id = ?";
        
        sqlite3_stmt* stmt;
        if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) == SQLITE_OK) {
            sqlite3_bind_int(stmt, 1, ruleId);
            sqlite3_step(stmt);
            sqlite3_finalize(stmt);
            reload_rules();
        }
    }
    
    void update_rule(const ClassificationRule& rule) {
        const char* sql = "UPDATE classification_rules SET "
                         "name = ?, description = ?, enabled = ?, priority = ?, "
                         "condition_field = ?, condition_operator = ?, condition_value = ?, "
                         "action_type = ?, action_classification = ?, updated_at = CURRENT_TIMESTAMP "
                         "WHERE id = ?";
        
        sqlite3_stmt* stmt;
        if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) == SQLITE_OK) {
            sqlite3_bind_text(stmt, 1, rule.name.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 2, rule.description.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_int(stmt, 3, rule.enabled ? 1 : 0);
            sqlite3_bind_int(stmt, 4, rule.priority);
            sqlite3_bind_text(stmt, 5, rule.condition_field.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 6, rule.condition_operator.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 7, rule.condition_value.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 8, rule.action_type.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_text(stmt, 9, rule.action_classification.c_str(), -1, SQLITE_STATIC);
            sqlite3_bind_int(stmt, 10, rule.id);
            
            if (sqlite3_step(stmt) == SQLITE_DONE) {
                reload_rules();
            }
            sqlite3_finalize(stmt);
        }
    }
    
    std::vector<ClassificationRule> get_all_rules() {
        std::lock_guard<std::mutex> lock(rulesMutex);
        return rules;
    }
};
```

### 2.4 Integration into Main Classification Flow

```cpp
// FILE: src/ClassificationEngine.cpp (UPDATED)

class ClassificationEngine {
private:
    Phase0_FastFilter phase0;
    Phase1_PreAnalysis phase1;
    Phase2_Patterns phase2;
    Phase25_AISidecar phase25;
    Phase3_Context phase3;
    Phase4_Decision phase4;
    RuleEngine ruleEngine;  // NEW
    
public:
    ClassificationResult classify(const std::string& filePath) {
        auto startTime = std::chrono::high_resolution_clock::now();
        
        // PHASES 0-4: Standard classification (as before)
        size_t fileSize = get_file_size(filePath);
        auto phase0Result = phase0.process(filePath, fileSize);
        
        ClassificationResult phaseResult;
        if (phase0Result.decided) {
            phaseResult = create_result_from_phase0(phase0Result);
        } else {
            // Continue with phases 1-4...
            phaseResult = full_classification_phases(filePath);
        }
        
        // [NEW IN V3] PHASE 5: RULE ENGINE EVALUATION
        auto ruleResult = ruleEngine.evaluate(filePath, phaseResult);
        
        if (ruleResult.matched) {
            // Rule was triggered - override Phase 0-4 result
            phaseResult.classification = ruleResult.new_classification;
            phaseResult.score = 100.0f;  // Rule-based classification is 100% confident
            phaseResult.confidence = 100.0f;
            phaseResult.explanation = ruleResult.explanation;
            phaseResult.ruleTriggered = ruleResult.rule_name;
            
            // Log rule trigger
            log_rule_trigger(filePath, phaseResult.classification, ruleResult.rule_id);
        }
        
        phaseResult.elapsedMs = elapsed_ms(startTime);
        return phaseResult;
    }
};
```

---

## PART 3: CLASSIFICATION RULES MANAGEMENT UI (NEW FOR V3)

### 3.1 Backend API Endpoints

```typescript
// FILE: backend/src/routes/rules.ts

import express from 'express';
import sqlite3 from 'sqlite3';
import { authenticate, authorize } from './middleware/auth';

const router = express.Router();
const db = new sqlite3.Database('./data/classifications.db');

// GET all rules
router.get('/api/rules', authenticate, (req, res) => {
    db.all(
        `SELECT id, name, description, enabled, priority, 
                condition_field, condition_operator, condition_value,
                action_type, action_classification, created_by, created_at
         FROM classification_rules 
         ORDER BY priority ASC`,
        (err, rows) => {
            if (err) {
                res.status(500).json({ error: err.message });
            } else {
                res.json(rows);
            }
        }
    );
});

// POST create new rule
router.post('/api/rules', authenticate, authorize('admin'), (req, res) => {
    const {
        name,
        description,
        enabled,
        priority,
        condition_field,
        condition_operator,
        condition_value,
        action_type,
        action_classification
    } = req.body;

    // Validate required fields
    if (!name || !condition_field || !condition_operator || !condition_value) {
        return res.status(400).json({ error: 'Missing required fields' });
    }

    db.run(
        `INSERT INTO classification_rules 
         (name, description, enabled, priority, condition_field, condition_operator, 
          condition_value, action_type, action_classification, created_by)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
        [name, description, enabled ? 1 : 0, priority || 100, condition_field, 
         condition_operator, condition_value, action_type, action_classification, req.user.username],
        function(err) {
            if (err) {
                res.status(500).json({ error: err.message });
            } else {
                // Notify C++ agent to reload rules
                notify_cpp_agent_reload_rules();
                res.json({ id: this.lastID, success: true });
            }
        }
    );
});

// PUT update rule
router.put('/api/rules/:id', authenticate, authorize('admin'), (req, res) => {
    const { id } = req.params;
    const {
        name,
        description,
        enabled,
        priority,
        condition_field,
        condition_operator,
        condition_value,
        action_type,
        action_classification
    } = req.body;

    db.run(
        `UPDATE classification_rules SET 
         name = ?, description = ?, enabled = ?, priority = ?,
         condition_field = ?, condition_operator = ?, condition_value = ?,
         action_type = ?, action_classification = ?, updated_at = CURRENT_TIMESTAMP
         WHERE id = ?`,
        [name, description, enabled ? 1 : 0, priority, condition_field,
         condition_operator, condition_value, action_type, action_classification, id],
        (err) => {
            if (err) {
                res.status(500).json({ error: err.message });
            } else {
                notify_cpp_agent_reload_rules();
                res.json({ success: true });
            }
        }
    );
});

// DELETE rule
router.delete('/api/rules/:id', authenticate, authorize('admin'), (req, res) => {
    const { id } = req.params;

    db.run(
        `DELETE FROM classification_rules WHERE id = ?`,
        [id],
        (err) => {
            if (err) {
                res.status(500).json({ error: err.message });
            } else {
                notify_cpp_agent_reload_rules();
                res.json({ success: true });
            }
        }
    );
});

// Test a rule against a file
router.post('/api/rules/test', authenticate, (req, res) => {
    const { filePath, ruleId } = req.body;

    // Call C++ agent to evaluate rule against file
    evaluate_rule_on_file(filePath, ruleId, (result) => {
        res.json(result);
    });
});

function notify_cpp_agent_reload_rules() {
    // Send HTTP/gRPC call to C++ agent to reload rules
    // POST http://localhost:9090/reload-rules
}

function evaluate_rule_on_file(filePath, ruleId, callback) {
    // Send HTTP call to C++ agent
    // POST http://localhost:9090/test-rule with {filePath, ruleId}
}

export default router;
```

### 3.2 Frontend React Component - Rules Management

```typescript
// FILE: frontend/src/components/ClassificationRules.tsx

import React, { useState, useEffect } from 'react';
import { Plus, Trash2, Edit2, AlertCircle } from 'lucide-react';

interface ClassificationRule {
    id?: number;
    name: string;
    description: string;
    enabled: boolean;
    priority: number;
    condition_field: string;
    condition_operator: string;
    condition_value: string;
    action_type: string;
    action_classification: string;
    created_by?: string;
    created_at?: string;
}

interface AddRuleModalProps {
    isOpen: boolean;
    onClose: () => void;
    onSave: (rule: ClassificationRule) => void;
    editingRule?: ClassificationRule;
}

// Modal Component
function AddRuleModal({ isOpen, onClose, onSave, editingRule }: AddRuleModalProps) {
    const [formData, setFormData] = useState<ClassificationRule>(
        editingRule || {
            name: '',
            description: '',
            enabled: true,
            priority: 100,
            condition_field: 'keyword',
            condition_operator: 'contains',
            condition_value: '',
            action_type: 'classify_as',
            action_classification: 'PRIVATE'
        }
    );

    const conditionFields = [
        { value: 'keyword', label: 'File Name Keyword' },
        { value: 'file_extension', label: 'File Extension' },
        { value: 'file_size', label: 'File Size (MB)' },
        { value: 'directory_path', label: 'Directory Path' },
        { value: 'content_pattern', label: 'Content Pattern' },
        { value: 'user_group', label: 'User Group' }
    ];

    const operators = {
        'keyword': ['equals', 'contains', 'matches_regex'],
        'file_extension': ['equals', 'in_list'],
        'file_size': ['gt', 'lt', 'equals'],
        'directory_path': ['equals', 'contains', 'matches_regex'],
        'content_pattern': ['contains', 'matches_regex'],
        'user_group': ['equals', 'in_list']
    };

    const classifications = ['PUBLIC', 'PRIVATE', 'CONFIDENTIAL', 'RESTRICTED'];

    if (!isOpen) return null;

    return (
        <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
            <div className="bg-white rounded-lg shadow-lg max-w-2xl w-full mx-4 max-h-[80vh] overflow-y-auto">
                {/* Header */}
                <div className="sticky top-0 bg-white border-b border-gray-200 px-6 py-4 flex justify-between items-center">
                    <h2 className="text-xl font-semibold text-gray-900">
                        {editingRule ? 'Edit Classification Rule' : 'Add Classification Rule'}
                    </h2>
                    <button
                        onClick={onClose}
                        className="text-gray-500 hover:text-gray-700"
                    >
                        ✕
                    </button>
                </div>

                {/* Body */}
                <div className="p-6 space-y-4">
                    {/* Rule Name */}
                    <div>
                        <label className="block text-sm font-medium text-gray-700 mb-1">
                            Rule Name
                        </label>
                        <input
                            type="text"
                            value={formData.name}
                            onChange={(e) => setFormData({ ...formData, name: e.target.value })}
                            placeholder="e.g., Payroll Files Classification"
                            className="w-full px-3 py-2 bg-white border border-gray-300 rounded-lg text-gray-900 focus:outline-none focus:ring-2 focus:ring-[#fd382f] focus:border-[#fd382f]"
                        />
                    </div>

                    {/* Description */}
                    <div>
                        <label className="block text-sm font-medium text-gray-700 mb-1">
                            Description
                        </label>
                        <textarea
                            value={formData.description}
                            onChange={(e) => setFormData({ ...formData, description: e.target.value })}
                            placeholder="Explain what this rule does..."
                            className="w-full px-3 py-2 bg-white border border-gray-300 rounded-lg text-gray-900 focus:outline-none focus:ring-2 focus:ring-[#fd382f] focus:border-[#fd382f]"
                            rows={2}
                        />
                    </div>

                    {/* Priority */}
                    <div>
                        <label className="block text-sm font-medium text-gray-700 mb-1">
                            Priority (0-1000, lower = higher)
                        </label>
                        <input
                            type="number"
                            value={formData.priority}
                            onChange={(e) => setFormData({ ...formData, priority: parseInt(e.target.value) })}
                            min="0"
                            max="1000"
                            className="w-full px-3 py-2 bg-white border border-gray-300 rounded-lg text-gray-900 focus:outline-none focus:ring-2 focus:ring-[#fd382f] focus:border-[#fd382f]"
                        />
                    </div>

                    {/* Enable Toggle */}
                    <div className="flex items-center">
                        <input
                            type="checkbox"
                            id="enabled"
                            checked={formData.enabled}
                            onChange={(e) => setFormData({ ...formData, enabled: e.target.checked })}
                            className="w-4 h-4 rounded border-gray-300"
                        />
                        <label htmlFor="enabled" className="ml-2 text-sm font-medium text-gray-700">
                            Enable this rule
                        </label>
                    </div>

                    {/* Condition Section */}
                    <div className="border-t border-gray-200 pt-4">
                        <h3 className="text-sm font-semibold text-gray-900 mb-3">Condition</h3>
                        <div className="grid grid-cols-3 gap-3">
                            {/* Condition Field */}
                            <div>
                                <label className="block text-xs text-gray-600 mb-1">Field</label>
                                <select
                                    value={formData.condition_field}
                                    onChange={(e) => setFormData({ ...formData, condition_field: e.target.value })}
                                    className="w-full px-2 py-1.5 bg-white border border-gray-300 rounded text-sm text-gray-900 focus:outline-none focus:ring-2 focus:ring-[#fd382f]"
                                >
                                    {conditionFields.map((field) => (
                                        <option key={field.value} value={field.value}>
                                            {field.label}
                                        </option>
                                    ))}
                                </select>
                            </div>

                            {/* Condition Operator */}
                            <div>
                                <label className="block text-xs text-gray-600 mb-1">Operator</label>
                                <select
                                    value={formData.condition_operator}
                                    onChange={(e) => setFormData({ ...formData, condition_operator: e.target.value })}
                                    className="w-full px-2 py-1.5 bg-white border border-gray-300 rounded text-sm text-gray-900 focus:outline-none focus:ring-2 focus:ring-[#fd382f]"
                                >
                                    {operators[formData.condition_field as keyof typeof operators]?.map((op) => (
                                        <option key={op} value={op}>
                                            {op === 'equals' && 'equals'}
                                            {op === 'contains' && 'contains'}
                                            {op === 'matches_regex' && 'matches regex'}
                                            {op === 'gt' && 'greater than'}
                                            {op === 'lt' && 'less than'}
                                            {op === 'in_list' && 'in list'}
                                        </option>
                                    ))}
                                </select>
                            </div>

                            {/* Condition Value */}
                            <div>
                                <label className="block text-xs text-gray-600 mb-1">Value</label>
                                <input
                                    type="text"
                                    value={formData.condition_value}
                                    onChange={(e) => setFormData({ ...formData, condition_value: e.target.value })}
                                    placeholder={
                                        formData.condition_field === 'file_size'
                                            ? 'e.g., 100 (for 100MB)'
                                            : formData.condition_field === 'file_extension'
                                            ? 'e.g., .xlsx, .sql'
                                            : 'e.g., payroll'
                                    }
                                    className="w-full px-2 py-1.5 bg-white border border-gray-300 rounded text-sm text-gray-900 focus:outline-none focus:ring-2 focus:ring-[#fd382f]"
                                />
                            </div>
                        </div>
                    </div>

                    {/* Action Section */}
                    <div className="border-t border-gray-200 pt-4">
                        <h3 className="text-sm font-semibold text-gray-900 mb-3">Action</h3>
                        <div>
                            <label className="block text-xs text-gray-600 mb-2">Classify As</label>
                            <div className="grid grid-cols-4 gap-2">
                                {classifications.map((classification) => (
                                    <button
                                        key={classification}
                                        onClick={() => setFormData({ ...formData, action_classification: classification })}
                                        className={`px-3 py-2 rounded text-sm font-medium transition-colors ${
                                            formData.action_classification === classification
                                                ? classification === 'PUBLIC'
                                                    ? 'bg-green-500/20 text-green-400 border border-green-500/30'
                                                    : classification === 'PRIVATE'
                                                    ? 'bg-blue-500/10 text-blue-400 border border-blue-500/30'
                                                    : classification === 'CONFIDENTIAL'
                                                    ? 'bg-yellow-500/10 text-yellow-400 border border-yellow-500/30'
                                                    : 'bg-red-500/10 text-red-400 border border-red-500/30'
                                                : 'bg-gray-100 text-gray-700 border border-gray-300'
                                        }`}
                                    >
                                        {classification}
                                    </button>
                                ))}
                            </div>
                        </div>
                    </div>

                    {/* Example */}
                    <div className="bg-blue-50 border border-blue-200 rounded-lg p-3">
                        <div className="flex gap-2 text-sm">
                            <AlertCircle className="w-4 h-4 text-blue-600 flex-shrink-0 mt-0.5" />
                            <div>
                                <p className="text-blue-900 font-medium">Example</p>
                                <p className="text-blue-800 text-xs mt-1">
                                    This rule will match files where "{formData.condition_field}" {formData.condition_operator} "{formData.condition_value}" and classify them as {formData.action_classification}.
                                </p>
                            </div>
                        </div>
                    </div>
                </div>

                {/* Footer */}
                <div className="sticky bottom-0 bg-gray-50 border-t border-gray-200 px-6 py-3 flex justify-end gap-3">
                    <button
                        onClick={onClose}
                        className="px-4 py-2 text-gray-700 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors"
                    >
                        Cancel
                    </button>
                    <button
                        onClick={() => {
                            onSave(formData);
                            onClose();
                        }}
                        className="px-4 py-2 bg-[#fd382f] text-white rounded-lg hover:bg-[#e02f26] transition-colors"
                    >
                        Save Rule
                    </button>
                </div>
            </div>
        </div>
    );
}

// Main Rules Component
export function ClassificationRules() {
    const [rules, setRules] = useState<ClassificationRule[]>([]);
    const [isModalOpen, setIsModalOpen] = useState(false);
    const [editingRule, setEditingRule] = useState<ClassificationRule | undefined>();
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        // Load rules on mount
        fetch('/api/rules')
            .then((res) => res.json())
            .then((data) => {
                setRules(data);
                setLoading(false);
            })
            .catch((err) => {
                console.error('Failed to load rules:', err);
                setLoading(false);
            });
    }, []);

    const handleAddRule = () => {
        setEditingRule(undefined);
        setIsModalOpen(true);
    };

    const handleEditRule = (rule: ClassificationRule) => {
        setEditingRule(rule);
        setIsModalOpen(true);
    };

    const handleSaveRule = async (formData: ClassificationRule) => {
        try {
            const method = editingRule ? 'PUT' : 'POST';
            const url = editingRule ? `/api/rules/${editingRule.id}` : '/api/rules';

            const response = await fetch(url, {
                method,
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(formData)
            });

            if (response.ok) {
                // Reload rules
                const updatedRules = await fetch('/api/rules').then((res) => res.json());
                setRules(updatedRules);
            }
        } catch (err) {
            console.error('Failed to save rule:', err);
        }
    };

    const handleDeleteRule = async (ruleId: number) => {
        if (!confirm('Delete this rule?')) return;

        try {
            const response = await fetch(`/api/rules/${ruleId}`, { method: 'DELETE' });

            if (response.ok) {
                setRules((prev) => prev.filter((r) => r.id !== ruleId));
            }
        } catch (err) {
            console.error('Failed to delete rule:', err);
        }
    };

    const getClassificationColor = (classification: string) => {
        switch (classification) {
            case 'PUBLIC':
                return { bg: 'bg-green-500/10', text: 'text-green-400', border: 'border-green-500/30' };
            case 'PRIVATE':
                return { bg: 'bg-blue-500/10', text: 'text-blue-400', border: 'border-blue-500/30' };
            case 'CONFIDENTIAL':
                return { bg: 'bg-yellow-500/10', text: 'text-yellow-400', border: 'border-yellow-500/30' };
            case 'RESTRICTED':
                return { bg: 'bg-red-500/10', text: 'text-red-400', border: 'border-red-500/30' };
            default:
                return { bg: 'bg-gray-500/10', text: 'text-gray-400', border: 'border-gray-500/30' };
        }
    };

    return (
        <div className="p-6">
            {/* Header */}
            <div className="flex justify-between items-center mb-6">
                <div>
                    <h1 className="text-2xl font-semibold text-gray-900">Classification Rules</h1>
                    <p className="text-gray-600 text-sm mt-1">
                        Create custom rules to automatically classify files
                    </p>
                </div>
                <button
                    onClick={handleAddRule}
                    className="flex items-center gap-2 px-4 py-2 bg-[#fd382f] text-white rounded-lg hover:bg-[#e02f26] transition-colors"
                >
                    <Plus className="w-4 h-4" />
                    Add Rule
                </button>
            </div>

            {/* Add Rule Modal */}
            <AddRuleModal
                isOpen={isModalOpen}
                onClose={() => {
                    setIsModalOpen(false);
                    setEditingRule(undefined);
                }}
                onSave={handleSaveRule}
                editingRule={editingRule}
            />

            {/* Rules List */}
            {loading ? (
                <div className="text-center py-12 text-gray-500">Loading rules...</div>
            ) : rules.length === 0 ? (
                <div className="bg-white rounded-lg border border-gray-200 p-8 text-center">
                    <p className="text-gray-500">No classification rules yet.</p>
                    <button
                        onClick={handleAddRule}
                        className="mt-4 px-4 py-2 bg-[#fd382f] text-white rounded-lg hover:bg-[#e02f26] transition-colors"
                    >
                        Create Your First Rule
                    </button>
                </div>
            ) : (
                <div className="space-y-3">
                    {rules.map((rule) => {
                        const colors = getClassificationColor(rule.action_classification);
                        return (
                            <div
                                key={rule.id}
                                className={`bg-white border border-gray-200 rounded-lg p-4 hover:shadow-md transition-shadow ${
                                    !rule.enabled ? 'opacity-60' : ''
                                }`}
                            >
                                <div className="flex items-start justify-between">
                                    <div className="flex-1">
                                        <div className="flex items-center gap-2 mb-2">
                                            <h3 className="font-semibold text-gray-900">{rule.name}</h3>
                                            {!rule.enabled && (
                                                <span className="px-2 py-0.5 bg-gray-200 text-gray-600 text-xs rounded font-medium">
                                                    Disabled
                                                </span>
                                            )}
                                            <span className="text-xs text-gray-500">Priority: {rule.priority}</span>
                                        </div>
                                        <p className="text-sm text-gray-600 mb-3">{rule.description}</p>

                                        <div className="grid grid-cols-2 gap-4 text-sm">
                                            <div>
                                                <span className="text-gray-600">Condition: </span>
                                                <code className="bg-gray-100 px-2 py-1 rounded text-xs">
                                                    {rule.condition_field} {rule.condition_operator} "{rule.condition_value}"
                                                </code>
                                            </div>
                                            <div>
                                                <span className="text-gray-600">Action: </span>
                                                <span
                                                    className={`inline-flex items-center font-medium rounded border px-2 py-0.5 text-xs ${colors.bg} ${colors.text} ${colors.border}`}
                                                >
                                                    {rule.action_classification}
                                                </span>
                                            </div>
                                        </div>
                                    </div>

                                    <div className="flex gap-2 ml-4">
                                        <button
                                            onClick={() => handleEditRule(rule)}
                                            className="p-2 text-gray-600 hover:text-[#fd382f] hover:bg-red-50 rounded transition-colors"
                                            title="Edit rule"
                                        >
                                            <Edit2 className="w-4 h-4" />
                                        </button>
                                        <button
                                            onClick={() => rule.id && handleDeleteRule(rule.id)}
                                            className="p-2 text-gray-600 hover:text-red-600 hover:bg-red-50 rounded transition-colors"
                                            title="Delete rule"
                                        >
                                            <Trash2 className="w-4 h-4" />
                                        </button>
                                    </div>
                                </div>
                            </div>
                        );
                    })}
                </div>
            )}
        </div>
    );
}
```

---

## PART 4: UPDATED EVENT LOGS INTEGRATION

The event logs dashboard now shows which rule triggered (if any):

```typescript
// FILE: frontend/src/components/EventLogs.tsx (UPDATED)

<tr key={idx}>
    <td>file_created</td>
    <td>{event.file_path.split('\\\\').pop()}</td>
    <td>{event.file_path}</td>
    <td>DESKTOP-DVJ9RNL$</td>
    <td style={{ color: getClassificationColor(event.classification), fontWeight: 'bold' }}>
        {event.classification} ({event.score.toFixed(0)}/100)
    </td>
    <td>{event.score.toFixed(1)}%</td>
    <td>{event.confidence.toFixed(0)}%</td>
    <td>
        {event.ruleTriggered ? (
            <span style={{ fontSize: '12px', color: '#fd382f' }} title={event.ruleTriggered}>
                Rule: {event.ruleTriggered}
            </span>
        ) : (
            <span style={{ fontSize: '12px', color: '#6b7280' }}>
                Auto-classified
            </span>
        )}
    </td>
    <td>{event.elapsed_ms}ms</td>
    <td>{new Date(event.created_at).toLocaleTimeString()}</td>
</tr>
```

---

## PART 5: COMPLETE IMPLEMENTATION CHECKLIST (UPDATED FOR V3)

### Phase 1: Database & Backend (Days 1-2)

- [ ] Add `classification_rules` table to SQLite schema
- [ ] Add `rule_audit_log` table
- [ ] Implement Rule CRUD API endpoints (GET, POST, PUT, DELETE /api/rules)
- [ ] Add rule validation logic
- [ ] Implement rule reload notification system

### Phase 2: C++ Rule Engine (Days 3-5)

- [ ] Implement `RuleEngine` class
- [ ] Add `evaluate_condition()` for all field types:
  - [ ] keyword (filename)
  - [ ] file_extension
  - [ ] file_size
  - [ ] directory_path
  - [ ] content_pattern
  - [ ] user_group
- [ ] Add `evaluate_operator()` for all operators:
  - [ ] equals, contains, matches_regex, gt, lt, in_list
- [ ] Integrate Phase 5 into ClassificationEngine
- [ ] Add rule trigger audit logging
- [ ] Implement rule reload on admin changes

### Phase 3: Frontend UI (Days 1-2 of Week 2)

- [ ] Create `AddRuleModal` component
- [ ] Implement rule form with validation
- [ ] Create `ClassificationRules` list component
- [ ] Add edit/delete functionality
- [ ] Add real-time WebSocket updates for rules

### Phase 4: Testing & Integration (Days 3-5 of Week 2)

- [ ] Unit tests for RuleEngine
  - [ ] Test each condition field
  - [ ] Test each operator
  - [ ] Test priority ordering
  - [ ] Test rule enable/disable
- [ ] Integration tests
  - [ ] File creation → Rule evaluation → Dashboard update
  - [ ] Add rule → Agent reloads → Applies to new files
  - [ ] Delete rule → Agent reloads → No longer applied
- [ ] Performance testing
  - [ ] Rule evaluation latency (<5ms even with 50 rules)
  - [ ] Database reload latency (<100ms)

### Phase 5: Production Hardening (Days 1-3 of Week 3)

- [ ] Add rule validation & sanitization
- [ ] Implement rule conflict detection
- [ ] Add rule audit trail
- [ ] Implement rule rollback capability
- [ ] Add rule testing endpoint (/api/rules/:id/test)
- [ ] Performance monitoring for rules

---

## PART 6: EXAMPLE SCENARIOS (NEW FOR V3)

### Scenario 1: File Created with Keyword Match

```
User creates: C:\\Finance\\2024_payroll.xlsx

T+0ms:   User creates file
T+3ms:   Phase 0-4 classification: PUBLIC (score 15)
T+10ms:  Phase 5: Rule Engine evaluates
         Rule 1 "Payroll Files": 
           - Condition: keyword = "payroll"
           - File contains "payroll" ✓ MATCH
           - Action: Classify as RESTRICTED
T+12ms:  Final result: RESTRICTED (score 100, ruleTriggered: "Payroll Files")
T+15ms:  WebSocket push to dashboard
T+20ms:  Dashboard displays: RESTRICTED (100/100) with Rule badge
```

### Scenario 2: File Size Rule

```
User creates: C:\\backup\\database_backup.sql (150MB)

T+0ms:   User initiates large file transfer
T+5ms:   Phase 0-4: INTERNAL (score 45)
T+15ms:  Phase 5: Rule Engine evaluates
         Rule 4 "Large Backups":
           - Condition: file_size > 100 MB
           - File size = 150MB ✓ MATCH
           - Action: Classify as RESTRICTED
T+18ms:  Final result: RESTRICTED (score 100, ruleTriggered: "Large Backups")
T+25ms:  Dashboard shows quarantine status
```

### Scenario 3: Directory-Based Classification

```
User creates: C:\\Marketing\\campaign_plan.pptx

T+0ms:   User creates file in Marketing folder
T+4ms:   Phase 0-4: CONFIDENTIAL (score 72)
T+12ms:  Phase 5: Rule Engine evaluates
         Rule 2 "Marketing Department":
           - Condition: directory_path contains "C:\\Marketing"
           - Directory path matches ✓ MATCH
           - Action: Classify as CONFIDENTIAL
T+15ms:  Final result: CONFIDENTIAL (score 100, ruleTriggered: "Marketing Department")
         (No change from Phase 0-4, but rule confirmed classification)
```

---

## PART 7: CRITICAL IMPLEMENTATION NOTES

### Performance Requirements (V3 Update)

```
✓ Phase 0-4 classification: <50ms
✓ Phase 5 (Rule Engine): <5ms
  - Load 50 rules: <1ms
  - Evaluate 50 rules: <4ms
✓ Total classification: <60ms (still invisible to user)
✓ Rule reload: <100ms
✓ Dashboard update: <150ms total
```

### Rule Priority System

```
Priority determines which rule fires first:
  0-99:    System default rules (do not modify)
  100:     High-priority user rules
  500:     Medium-priority user rules
  1000:    Low-priority user rules

First matching rule wins (no chaining).
Example:
  Rule "Payroll" (priority 10) matches
  Rule "Finance Files" (priority 50) would match
  → Only "Payroll" fires
```

### Regex Safety

```
⚠️  IMPORTANT: All regex patterns must be validated before storing:
    ✓ Compile regex before saving rule
    ✓ Test regex against sample text
    ✓ Add timeout (max 5ms per regex evaluation)
    ✓ Reject invalid regex patterns

Example:
  Invalid: "(?P<name>..." → Reject
  Valid: "^payroll_" → Accept
```

---

## PART 8: API CONTRACT (For AI Coding Agent)

### What You Must Implement

**C++ Side:**
```
1. RuleEngine class with:
   - evaluate(filePath, classificationResult) → RuleEvaluationResult
   - add_rule(rule) → void
   - delete_rule(id) → void
   - update_rule(rule) → void
   - reload_rules() → void
   
2. Database schema with classification_rules table

3. Condition evaluators for all field types
```

**Backend Side:**
```
1. GET /api/rules → [{id, name, condition_field, ...}]
2. POST /api/rules → {success: true, id: number}
3. PUT /api/rules/:id → {success: true}
4. DELETE /api/rules/:id → {success: true}
5. POST /api/rules/test → {matches: true, newClassification: "RESTRICTED"}
```

**Frontend Side:**
```
1. <ClassificationRules /> component
2. AddRuleModal with form
3. Rule list with edit/delete buttons
4. Integration with event logs dashboard
```

---

## PART 9: FINAL CHECKLIST FOR AI AGENT

**DO NOT STOP CODING UNTIL:**

- ✅ Database schema created with rules table
- ✅ C++ RuleEngine fully implemented (all 6 field types, all 6 operators)
- ✅ Phase 5 integrated into ClassificationEngine
- ✅ Backend APIs working (CRUD operations)
- ✅ Frontend components rendering correctly
- ✅ Rules persist across restarts
- ✅ Rule reload <100ms
- ✅ Phase 5 evaluation <5ms (even with 50 rules)
- ✅ Empty file + "payroll" rule → RESTRICTED
- ✅ Dashboard shows rule name when triggered
- ✅ Add rule → Auto-applies to next files
- ✅ Delete rule → Stops applying
- ✅ Edit rule → Saves immediately
- ✅ Priority ordering works correctly
- ✅ Regex validation prevents crashes
- ✅ Unit tests pass (>90% coverage)
- ✅ Integration tests pass (end-to-end)
- ✅ Performance targets met (<60ms total)
- ✅ No TODOs in code
- ✅ Production monitoring active
- ✅ Deployed as Windows service

**Definition of DONE:**
- Code compiles without warnings
- All tests green
- Latency <60ms (Phase 0-4: 50ms + Phase 5: 10ms buffer)
- Rules work immediately after creation
- Dashboard updates in real-time
- Zero crashes on edge cases
- Production-ready for deployment

---

## APPENDIX: Referenced Documents

**See CLASSIFICATION-MATRIX-V2.md for:**
- Detailed scoring rules (Section 1.1)
- Regex patterns (Section 2.2)
- Validator functions (Section 4.0)

**See IMPLEMENTATION-GUIDE-V2.md for:**
- Phase 0-4 C++ source (Section 2)
- Testing examples (Section 4)
- Deployment checklist (Section 5)

**See QUICK-REFERENCE-V2.md for:**
- Score ranges
- Edge cases
- Common pitfalls

---

*Generated for PRITRAK V3.0 Real-Time Classification System with Rule Engine*  
*Status: PRODUCTION-READY SPECIFICATION (COMPLETE)*  
*Date: January 11, 2026*  
*Next Phase: AI Coding Agent Implementation*
