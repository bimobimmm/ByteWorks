# Oracle Housekeeping Guide

> *A practical guide to manage and clean Oracle audit, log, and trace files manually*

---

### Overview
> This document outlines a manual approach to Oracle housekeeping,
> allowing controlled cleanup of audit, log, and trace files by
> navigating directories step-by-step before executing removal.

---

### Scope
> • Audit files (.aud)  
> • Log files (.log)  
> • Trace files (.trc)  

---

### Step 1 — Identify Base Directory
> Enter Oracle base directory:

```
cd $ORACLE_BASE
```

---

### Step 2 — Locate Audit Directory
> Navigate to audit file location:

```
cd admin/$ORACLE_SID/adump
```

> Verify files:

```
ls -lh
```

---

### Step 3 — Cleanup Audit Files (.aud)
> Remove files older than 7 days:

```
find . -type f -name "*.aud" -mtime +7 -exec rm -f {} \;
```

---

### Step 4 — Locate Trace Directory
> Navigate step-by-step to trace directory:

```
cd $ORACLE_BASE/diag/rdbms
ls
cd <db_name>
cd <instance_name>/trace
```

> Verify files:

```
ls -lh *.trc
ls -lh *.log
```

---

### Step 5 — Cleanup Trace Files (.trc)
```
find . -type f -name "*.trc" -mtime +7 -exec rm -f {} \;
```

---

### Step 6 — Cleanup Log Files (.log)
```
find . -type f -name "*.log" -mtime +7 -exec rm -f {} \;
```

---

### Step 7 — Optional Preview (Recommended)
> Review files before deletion:

```
find . -type f -mtime +7
```

---

### Notes
> • Always navigate and verify directory before cleanup  
> • Avoid direct execution without checking file contents  
> • Adjust retention period based on system requirements  
> • Manual approach provides better control in production environments  

---

### Author
**noxiousroots**  
@ByteWorks  
