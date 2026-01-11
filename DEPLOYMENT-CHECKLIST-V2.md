# DEPLOYMENT CHECKLIST V2.0

**STATUS:** Production-Ready Implementation  
**VERSION:** 2.0  
**LAST UPDATED:** January 11, 2026  
**ESTIMATED TIME TO DEPLOY:** 4-6 hours  

---

## PRE-DEPLOYMENT (DEVELOPMENT)

### C++ Development

- [ ] All source files created (classification_engine.cpp, validators.cpp, pattern_matcher.cpp, etc.)
- [ ] CMakeLists.txt configured
- [ ] Dependencies installed via vcpkg (nlohmann-json, sqlite3, re2, websocketpp)
- [ ] Code compiles without warnings: `cmake --build . --config Release`
- [ ] No compiler errors
- [ ] All #include guards present
- [ ] No memory leaks (valgrind or Dr. Memory)

### Unit Tests

- [ ] Test suite created (test_classification.cpp, test_validators.cpp, test_performance.cpp)
- [ ] All validators tested:
  - [ ] Luhn algorithm (credit cards)
  - [ ] SSN validation
  - [ ] French NIR validation
  - [ ] IBAN validation
- [ ] Pattern matcher tested:
  - [ ] Credit card regex
  - [ ] SSN regex
  - [ ] French NIR regex
  - [ ] Email regex
- [ ] Classification engine tested:
  - [ ] Phase 0 fast filter
  - [ ] Phase 1 pre-analysis
  - [ ] Phase 2 content patterns
  - [ ] Phase 3 context logic
  - [ ] Phase 4 decision
- [ ] All tests pass: `ctest --output-on-failure`
- [ ] Code coverage > 85%

### Performance Testing

- [ ] Empty file classified in < 2ms (Phase 0)
- [ ] Phase 1 analysis < 5ms
- [ ] Phase 2 pattern matching < 30ms
- [ ] Overall classification < 50ms p95
- [ ] WebSocket push < 10ms
- [ ] Database insert < 5ms
- [ ] Memory usage < 200MB
- [ ] CPU < 10% per file
- [ ] False positive rate < 1.5% on test set
- [ ] False negative rate < 1%
- [ ] Accuracy > 95%

### Test Cases

**Test Set 1: Basic Files**
- [ ] Empty .txt file → PUBLIC (0/100) in <2ms
- [ ] Private key (.key) → RESTRICTED (100/100) in <2ms
- [ ] Log file (.log) → PUBLIC (0/100) in <2ms
- [ ] Env file (.env) → RESTRICTED (100/100) in <2ms

**Test Set 2: PII Detection**
- [ ] File with credit card (Luhn valid) → CONFIDENTIAL/RESTRICTED
- [ ] File with SSN → CONFIDENTIAL/RESTRICTED
- [ ] File with French NIR → RESTRICTED
- [ ] File with test credit card (negative context) → public/internal

**Test Set 3: Business Data**
- [ ] Payroll CSV with 2 employees → CONFIDENTIAL (50-89)
- [ ] Payroll CSV with 100 employees → RESTRICTED (90+)
- [ ] Customer list → INTERNAL/CONFIDENTIAL
- [ ] Invoice CSV → INTERNAL/CONFIDENTIAL

**Test Set 4: False Positives**
- [ ] Documentation with example credit card → Internal/public (with negative context)
- [ ] Code file with commented-out credentials → correct classification
- [ ] README with test data → public/internal

**Test Set 5: Edge Cases**
- [ ] Very large file (> 100MB) → fast classification (partial read)
- [ ] Binary file → public
- [ ] Encrypted file → can't read, classify as INTERNAL
- [ ] File with mixed encoding → handle gracefully

---

## DEPLOYMENT (PRODUCTION)

### Windows Environment Setup

- [ ] Windows Server 2019 or later (or Windows 10/11 for testing)
- [ ] .NET Framework 4.7+ installed
- [ ] Visual C++ Redistributables installed
- [ ] Administrator access for Windows service installation

### Directory Structure

```
C:\Program Files\PRITRAK\
├── pritrak_dlp.exe                    (Main executable)
├── config.json                        (Configuration)
├── vcruntime140.dll                   (Runtime dependencies)
└── msvcp140.dll

C:\ProgramData\PRITRAK\
├── classifications.db                 (SQLite database)
├── pritrak.log                        (Log file)
└── temp\                              (Temporary files)
```

- [ ] Create C:\\Program Files\\PRITRAK\\ directory
- [ ] Create C:\\ProgramData\\PRITRAK\\ directory
- [ ] Copy pritrak_dlp.exe to Program Files\\PRITRAK\\
- [ ] Copy runtime DLLs to Program Files\\PRITRAK\\

### Configuration Files

**C:\\Program Files\\PRITRAK\\config.json**

```json
{
  "usn_journal": {
    "enabled": true,
    "volume": "C:\\",
    "poll_interval_ms": 100,
    "exclude_patterns": [
      "Windows",
      "AppData",
      "Temp",
      "ProgramFiles",
      ".git"
    ]
  },
  "classification": {
    "phase0_enabled": true,
    "phase25_ai_enabled": false,
    "ai_api_url": "http://localhost:9000/classify",
    "ai_timeout_ms": 100
  },
  "websocket": {
    "enabled": true,
    "server_url": "ws://localhost:8080/ws/classification",
    "reconnect_interval_ms": 5000,
    "max_retries": 5
  },
  "database": {
    "path": "C:\\ProgramData\\PRITRAK\\classifications.db",
    "auto_vacuum": true,
    "retention_days": 90
  },
  "logging": {
    "level": "INFO",
    "file": "C:\\ProgramData\\PRITRAK\\pritrak.log",
    "max_size_mb": 100,
    "backup_count": 10
  },
  "performance": {
    "max_concurrent_classifications": 10,
    "file_read_timeout_ms": 5000,
    "max_file_size_bytes": 1000000
  }
}
```

- [ ] Create config.json
- [ ] Verify all paths exist
- [ ] Set appropriate permissions (read-only for service account)

### Database Setup

```sql
CREATE TABLE IF NOT EXISTS event_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    event_type TEXT NOT NULL,                          -- "file_created", "file_modified"
    file_path TEXT UNIQUE NOT NULL,
    file_size INTEGER,
    classification TEXT NOT NULL,                       -- "PUBLIC", "INTERNAL", "CONFIDENTIAL", "RESTRICTED"
    score REAL NOT NULL,                               -- 0-100
    confidence REAL NOT NULL,                          -- 0-100
    risk_level TEXT NOT NULL,                          -- "NONE", "LOW", "MEDIUM_HIGH", "CRITICAL"
    explanation TEXT,
    elapsed_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_file_path ON event_logs(file_path);
CREATE INDEX IF NOT EXISTS idx_classification ON event_logs(classification);
CREATE INDEX IF NOT EXISTS idx_created_at ON event_logs(created_at);
CREATE INDEX IF NOT EXISTS idx_risk_level ON event_logs(risk_level);
```

- [ ] Create initial SQLite database
- [ ] Initialize tables and indexes
- [ ] Test database connection

### Windows Service Installation

```bash
# Run as Administrator
sc create PRITRAK-DLP `
  binPath= "C:\\Program Files\\PRITRAK\\pritrak_dlp.exe" `
  start= auto `
  displayName= "PRITRAK Data Loss Prevention"

sc start PRITRAK-DLP
sc query PRITRAK-DLP  # Verify status
```

- [ ] Service created successfully
- [ ] Service starts automatically on reboot
- [ ] Service runs under Network Service account
- [ ] Event Viewer shows no errors

### Firewall Rules

- [ ] Allow pritrak_dlp.exe for network connections
- [ ] Allow local WebSocket connections (port 8080)
- [ ] Restrict outbound to backend only (if restricted network)

### Backend Service (Node.js)

**Verify running:**

```bash
npm install
npm run dev  # or npm start for production
```

- [ ] Backend listening on port 8080
- [ ] WebSocket endpoint available at ws://localhost:8080/ws/classification
- [ ] SQLite database connection working
- [ ] CORS configured for dashboard (port 5173)

- [ ] Test connection: `wscat -c ws://localhost:8080/ws/classification`

### Dashboard Verification

- [ ] Frontend running at http://localhost:5173
- [ ] Event logs page loads
- [ ] WebSocket connection established (check console)
- [ ] Can filter by classification
- [ ] Can filter by date range

---

## TESTING (PRODUCTION)

### Manual Testing

**Test 1: Empty File Creation**

```bash
# Command
echo . > C:\Users\PC\Desktop\test_empty.txt

# Expected
# Dashboard shows within 50ms:
# - event_type: file_created
# - file_path: C:\Users\PC\Desktop\test_empty.txt
# - classification: PUBLIC
# - score: 0
# - risk_level: NONE
```

- [ ] Empty file classified as PUBLIC
- [ ] Classification appears in dashboard < 50ms
- [ ] Score is 0
- [ ] Confidence is 100

**Test 2: Sensitive File**

```bash
# Create file with test SSN
echo "Employee,SSN" > C:\Users\PC\Desktop\test_payroll.csv
echo "Alice,123-45-6789" >> C:\Users\PC\Desktop\test_payroll.csv

# Expected
# Dashboard shows within 50ms:
# - classification: CONFIDENTIAL or RESTRICTED
# - score: 50-100
# - risk_level: MEDIUM_HIGH or CRITICAL
```

- [ ] File with SSN classified correctly
- [ ] Classification appears < 50ms
- [ ] Score is >= 50
- [ ] Risk level matches classification

**Test 3: File Modification**

```bash
# Modify with more data
echo "Bob,987-65-4321" >> C:\Users\PC\Desktop\test_payroll.csv

# Expected
# Dashboard updates with higher score
```

- [ ] Classification updates on file modification
- [ ] Score increases with more PII
- [ ] Database reflects change

**Test 4: Excluded Files**

```bash
# Create file in Windows directory (should be excluded)
echo test > C:\Windows\Temp\test_temp.txt

# Expected
# NO entry in dashboard (file excluded)
```

- [ ] System files are excluded
- [ ] Temp files are excluded
- [ ] No false positives in dashboard

### Stress Testing

- [ ] Create 100 files in 1 minute
  - [ ] All classified
  - [ ] No dropped events
  - [ ] Memory < 300MB
  - [ ] CPU < 25%

- [ ] Create 1000 files in 10 minutes
  - [ ] All classified
  - [ ] Database responsive
  - [ ] WebSocket keeps up
  - [ ] Dashboard remains responsive

### Regression Testing

Run full test suite again:

```bash
ctest --output-on-failure
```

- [ ] All unit tests pass
- [ ] Performance benchmarks met
- [ ] No new false positives

---

## MONITORING (PRODUCTION)

### Dashboard Metrics

**Daily Check:**

```sql
-- Check classification distribution
SELECT 
    classification,
    COUNT(*) as count,
    AVG(score) as avg_score
FROM event_logs
WHERE DATE(created_at) = CURDATE()
GROUP BY classification;

-- Check performance
SELECT 
    AVG(elapsed_ms) as avg_latency,
    MAX(elapsed_ms) as max_latency,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY elapsed_ms) as p95_latency
FROM event_logs
WHERE DATE(created_at) = CURDATE();
```

- [ ] Average latency < 30ms
- [ ] P95 latency < 50ms
- [ ] RESTRICTED files < 5% of total
- [ ] No classification errors

### Alert Thresholds

- [ ] **CRITICAL:** P95 latency > 100ms
- [ ] **CRITICAL:** Classification error rate > 1%
- [ ] **WARNING:** P95 latency > 75ms
- [ ] **WARNING:** RESTRICTED file count > 10% of daily total
- [ ] **INFO:** Service restart detected

### Log Monitoring

**File:** C:\\ProgramData\\PRITRAK\\pritrak.log

- [ ] Check for errors every morning
- [ ] Archive logs weekly
- [ ] Keep 10 backup files
- [ ] Monitor for patterns

**Common Issues:**

```
[ERROR] WebSocket connection failed
→ Check backend service is running
→ Check firewall rules
→ Check network connectivity

[ERROR] Failed to read file
→ Check file permissions
→ Check file path is valid
→ Check file isn't locked by another process

[WARNING] Classification took 100ms
→ Check if file is very large
→ Check if system resources are low
→ Consider increasing thread pool size
```

### Automated Monitoring

Create monitoring script (monitor.ps1):

```powershell
# Check service status
$service = Get-Service -Name "PRITRAK-DLP"
if ($service.Status -ne "Running") {
    Write-Error "PRITRAK service is not running"
    Start-Service -Name "PRITRAK-DLP"
}

# Check database size
$dbSize = (Get-Item "C:\ProgramData\PRITRAK\classifications.db").Length / 1MB
if ($dbSize -gt 1000) {
    Write-Warning "Database size exceeded 1GB"
    # Run cleanup
}

# Check log file size
$logSize = (Get-Item "C:\ProgramData\PRITRAK\pritrak.log").Length / 1MB
if ($logSize -gt 100) {
    Write-Warning "Log file size exceeded 100MB"
    # Archive log
}
```

- [ ] Create scheduled task (runs every hour)
- [ ] Set to run with highest privileges
- [ ] Test execution

---

## MAINTENANCE

### Weekly

- [ ] Review classification accuracy metrics
- [ ] Check false positive/negative rates
- [ ] Review error logs
- [ ] Backup database: `copy C:\ProgramData\PRITRAK\classifications.db C:\Backups\classifications_$(Get-Date -Format yyyyMMdd).db`

### Monthly

- [ ] Archive old logs (>30 days)
- [ ] Review and update exclusion patterns if needed
- [ ] Test disaster recovery (restore from backup)
- [ ] Review classification rules (any updates?)
- [ ] Performance trend analysis

### Quarterly

- [ ] Update AI model if available
- [ ] Review false positive patterns
- [ ] Adjust scoring thresholds if needed
- [ ] Optimize database (VACUUM)
- [ ] Review and update documentation

### Database Maintenance

```sql
-- Clean up old entries (keep 90 days)
DELETE FROM event_logs WHERE created_at < datetime('now', '-90 days');

-- Optimize database
VACUUM;

-- Check database integrity
PRAGMA integrity_check;
```

- [ ] Schedule cleanup job monthly
- [ ] Monitor database growth
- [ ] Keep backups for 1 year

---

## SECURITY

### Access Control

- [ ] Restrict pritrak_dlp.exe execution to Network Service account
- [ ] Set C:\\Program Files\\PRITRAK\\ permissions:
  - [ ] Administrators: Full Control
  - [ ] Network Service: Read, Execute
  - [ ] Everyone: Deny
- [ ] Set C:\\ProgramData\\PRITRAK\\ permissions:
  - [ ] Administrators: Full Control
  - [ ] Network Service: Read, Write, Modify
  - [ ] Everyone: Deny

### Data Protection

- [ ] Enable Windows Defender for pritrak_dlp.exe
- [ ] Sign executable with code signing certificate (future)
- [ ] Enable audit logging for all file access
- [ ] Encrypt sensitive configuration (future)

### Network Security

- [ ] WebSocket connections are local only (localhost)
- [ ] Use WSS (WebSocket Secure) in production (future)
- [ ] Implement API authentication (future)
- [ ] Add rate limiting (future)

---

## ROLLBACK PLAN

If critical issues arise:

**Step 1: Stop Service**
```bash
sc stop PRITRAK-DLP
```

**Step 2: Restore Previous Version**
```bash
copy C:\\Backups\\pritrak_dlp_v1.exe C:\\Program Files\\PRITRAK\\pritrak_dlp.exe
```

**Step 3: Restore Database Backup**
```bash
copy C:\\Backups\\classifications_20260110.db C:\\ProgramData\\PRITRAK\\classifications.db
```

**Step 4: Start Service**
```bash
sc start PRITRAK-DLP
```

**Step 5: Verify**
- [ ] Service is running
- [ ] Dashboard shows data
- [ ] Classification works
- [ ] No errors in logs

---

## GO-LIVE SIGN-OFF

- [ ] All pre-deployment checks complete
- [ ] All unit tests pass
- [ ] All performance benchmarks met
- [ ] All manual tests pass
- [ ] Monitoring configured
- [ ] Backup plan tested
- [ ] Security review complete
- [ ] Documentation complete
- [ ] Team trained
- [ ] Go-live approved by management

**Approved by:** _______________  
**Date:** _______________  
**Go-live Date:** January 12, 2026  

---

## SUPPORT

**For issues:**
1. Check logs: C:\\ProgramData\\PRITRAK\\pritrak.log
2. Verify service: `sc query PRITRAK-DLP`
3. Test WebSocket: `wscat -c ws://localhost:8080/ws/classification`
4. Check database: Open classifications.db with SQLite Browser
5. Contact: security-team@pritrak.com

---

*Last Updated: January 11, 2026*  
*Version: 2.0 - Production Ready*
