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

### Notes & Common Errors

#### ORA-16475: succeeded with warnings
- Penyebab: standby belum memiliki standby redo log
- Status: **tidak blocking**, switchover tetap bisa dilakukan

##### ✔ Solusi (Recommended)

Tambahkan standby redo log di standby:

```sql
ALTER DATABASE ADD STANDBY LOGFILE SIZE 200M;
```

Cek existing log:

```sql
SELECT GROUP#, TYPE, MEMBER FROM V$LOGFILE;
```

---

#### ORA-16466 / ORA-16468 (standby not ready)
- Penyebab: standby belum fully synchronized

##### ✔ Solusi

Jalankan di primary:

```sql
ALTER SYSTEM ARCHIVE LOG CURRENT;
```

Lalu cek di standby:

```sql
SELECT PROCESS, STATUS FROM V$MANAGED_STANDBY;
```

Pastikan MRP status:
```
APPLYING_LOG
```

---

## 3. Execute Switchover

```sql
ALTER DATABASE SWITCHOVER TO ByteWorks_standby;
```

- Primary akan otomatis shutdown setelah selesai

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

Login ke database yang sekarang menjadi standby:

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT USING CURRENT LOGFILE;
```

---

### 8.2 Verify MRP Running

```sql
SELECT PROCESS, STATUS, THREAD#, SEQUENCE# FROM V$MANAGED_STANDBY;
```

Pastikan:
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

Pastikan arah archive sudah sesuai role baru.

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
- Error ORA-16475 tidak blocking, tapi tetap perlu perbaikan (standby redo log)  
- Setelah switchover:
  - Pastikan role benar
  - Pastikan MRP aktif
  - Pastikan tidak ada archive gap  
