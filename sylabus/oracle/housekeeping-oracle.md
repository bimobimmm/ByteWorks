# Oracle Housekeeping — Audit, Log, Trace (ByteWorks)

## Environment

| Parameter | Value |
|----------|------|
| ORACLE_BASE | /data/u01/app/oracle |
| ORACLE_SID | BYTEWORK |
| Audit Directory | $ORACLE_BASE/admin/$ORACLE_SID/adump |
| Trace Directory | $ORACLE_BASE/diag/rdbms/<db_name>/<instance_name>/trace |
| Retention (AUD) | 7 Days |
| Retention (LOG/TRC) | 7 Days |

---

### Audit File Cleanup

#### 1. Navigate to Audit Directory
```bash
cd $ORACLE_BASE/admin/$ORACLE_SID/adump
ls -lh
```

#### 2. Validate Files
```bash
ls -lh *.aud
```

#### 3. Cleanup Audit Files (.aud)
```bash
find . -type f -name "*.aud" -mtime +7 -exec rm -f {} \;
```

---

### Trace & Log Cleanup

#### 4. Navigate to Trace Directory
```bash
cd $ORACLE_BASE/diag/rdbms
ls
cd <db_name>
cd <instance_name>/trace
```

#### 5. Validate Files
```bash
ls -lh *.trc
ls -lh *.log
```

#### 6. Cleanup Trace Files (.trc)
```bash
find . -type f -name "*.trc" -mtime +7 -exec rm -f {} \;
```

#### 7. Cleanup Log Files (.log)
```bash
find . -type f -name "*.log" -mtime +7 -exec rm -f {} \;
```

---

### Validation & Safety

#### 8. Preview Before Delete (Recommended)
```bash
find . -type f -mtime +7
```

#### 9. Disk Usage Check
```bash
du -sh .
df -h
```

---

### Optional

#### 10. Compress Before Deletion
```bash
find . -type f -mtime +3 -exec gzip {} \;
```

---

## Summary

- Manual navigation ensures correct directory targeting  
- Cleanup is based on retention policy  
- Validation is performed before deletion  
- Flexible for different Oracle environments  
- Safe and suitable for production systems  
