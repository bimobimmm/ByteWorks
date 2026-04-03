# Oracle Data Guard Switchover Guide

## 1. Pre-Check

### 1.1 Check DB Unique Name

```sql
SHOW PARAMETER db_unique_name;
```

---

### 1.2 Check Alert Log Location

```bash
ls -ltr $ORACLE_BASE/diag/rdbms/*/*/trace/alert_*.log
```

---

## 2. Verify Standby Readiness

```sql
ALTER DATABASE SWITCHOVER TO ByteWorks_standby VERIFY;
```

---

## Notes & Common Errors

### ORA-16475: succeeded with warnings

- Penyebab: standby redo log (SRL) tidak ada atau tidak sesuai  
- Status: tidak blocking, tetapi **harus diperbaiki**

---

### Recommended Solution — Recreate Standby Redo Log

Pendekatan terbaik adalah **drop dan recreate standby redo log**, dengan ukuran yang sama seperti redo log primary.

---

#### Step 1 — Cek Redo Log Size (Primary)

```sql
SELECT GROUP#, BYTES/1024/1024 AS SIZE_MB FROM V$LOG;
```

---

#### Step 2 — Cek Standby Redo Log (Standby)

```sql
SELECT GROUP#, BYTES/1024/1024 AS SIZE_MB FROM V$STANDBY_LOG;
```

---

#### Step 3 — Drop Standby Redo Log Lama

```sql
ALTER DATABASE DROP STANDBY LOGFILE GROUP <GROUP#>;
```

---

#### Step 4 — Recreate Standby Redo Log

Jumlah minimal:
```
(Online Redo Log Group + 1)
```

Contoh:

```sql
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 '/data/oradata/ByteWorks/srl04.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 '/data/oradata/ByteWorks/srl05.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 '/data/oradata/ByteWorks/srl06.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 '/data/oradata/ByteWorks/srl07.log' SIZE 200M;
```

---

#### Step 5 — Verifikasi

```sql
SELECT GROUP#, STATUS, BYTES/1024/1024 AS SIZE_MB FROM V$STANDBY_LOG;
```

---

### Notes

- Size standby redo log harus sama atau lebih besar dari primary  
- Jumlah SRL ≥ (redo log group primary + 1)  
- Disarankan menggunakan ukuran yang sama untuk performa optimal  

---

### ORA-16466 / ORA-16468 (Standby Not Ready)

- Penyebab: standby belum sinkron

#### Solusi

```sql
ALTER SYSTEM ARCHIVE LOG CURRENT;
```

Cek status:

```sql
SELECT PROCESS, STATUS FROM V$MANAGED_STANDBY;
```

Pastikan:
```
MRP0 → APPLYING_LOG
```

---

## 3. Execute Switchover

```sql
ALTER DATABASE SWITCHOVER TO ByteWorks_standby;
```

- Primary akan otomatis shutdown

---

## 4. Startup Database (New Primary)

```sql
STARTUP;
```

---

## 5. Validate Role

```sql
SELECT NAME, OPEN_MODE, DATABASE_ROLE FROM V$DATABASE;
SELECT SWITCHOVER_STATUS FROM V$DATABASE;
SELECT DATABASE_ROLE, SWITCHOVER_STATUS FROM V$DATABASE;
SELECT NAME, DATABASE_ROLE, OPEN_MODE, SWITCHOVER_STATUS FROM V$DATABASE;
```

---

## 6. Check Replication Sync

```sql
SET LINESIZE 200
SET PAGESIZE 100
COLUMN name FORMAT A30
COLUMN value FORMAT A50

SELECT 
    d.name AS db_name,
    d.db_unique_name,
    d.database_role AS role,
    a.dest_id,
    a.thread#,
    b.last_seq,
    a.applied_seq,
    a.last_app_timestamp,
    b.last_seq - a.applied_seq AS arc_diff
FROM 
    (SELECT dest_id, thread#, MAX(sequence#) applied_seq,
            MAX(next_time) last_app_timestamp
     FROM v$archived_log 
     WHERE applied IN ('YES','IN-MEMORY')
     GROUP BY dest_id, thread#) a,
    (SELECT thread#, MAX(sequence#) last_seq 
     FROM v$archived_log 
     GROUP BY thread#) b,
    v$database d
WHERE a.thread# = b.thread#;
```

---

## 7. Test Log Shipping

```sql
ALTER SYSTEM ARCHIVE LOG CURRENT;
```

Pastikan:
```
ARC_DIFF = 0
```

---

## 8. Post-Switchover Fix (IMPORTANT)

### 8.1 Start MRP on New Standby

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT USING CURRENT LOGFILE;
```

---

### 8.2 Verify MRP

```sql
SELECT PROCESS, STATUS, THREAD#, SEQUENCE# FROM V$MANAGED_STANDBY;
```

Expected:
```
MRP0 → APPLYING_LOG
```

---

### 8.3 Verify Role

```sql
SELECT NAME, DATABASE_ROLE, OPEN_MODE FROM V$DATABASE;
```

Expected:
- Primary → READ WRITE  
- Standby → READ ONLY / MOUNTED  

---

### 8.4 Check Archive Destination

```sql
SHOW PARAMETER log_archive_dest;
```

---

## 9. System Validation

```bash
date
hostname
hostname -i
```

---

## Summary

- Switchover adalah proses planned tanpa data loss  
- Role primary dan standby akan bertukar  
- Error ORA-16475 tidak blocking, tetapi wajib diperbaiki  
- Standby redo log harus sesuai dengan primary  
- Setelah switchover:
  - Pastikan MRP aktif  
  - Pastikan tidak ada archive gap  
  - Pastikan role sudah benar  
