# Oracle Active Data Guard (ADG) Configuration Script

## 1. Environment

- ORACLE_SID       : BYTEWORK
- ORACLE_HOME      : /data/u01/app/oracle/product/19.0.0/dbhome_19
- DB_UNIQUE_NAME   : ByteWorks / ByteWorks_standby
- Archive Location : /data/archivelog/ByteWorks

---

# =========================================
# 2. ENABLE ARCHIVELOG (PRIMARY & STANDBY)
# =========================================

```sql
ARCHIVE LOG LIST;
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```

---

# =========================================
# 3. SET DB UNIQUE NAME
# =========================================

```sql
-- PRIMARY
ALTER SYSTEM SET db_unique_name='ByteWorks' SCOPE=BOTH;

-- STANDBY
ALTER SYSTEM SET db_unique_name='ByteWorks_standby' SCOPE=BOTH;
```

---

# =========================================
# 4. DATA GUARD CONFIGURATION
# =========================================

## PRIMARY

```sql
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(ByteWorks,ByteWorks_standby)' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1='LOCATION=/data/archivelog/ByteWorks VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=ByteWorks' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=BYTEWORK_STANDBY ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=ByteWorks_standby' SCOPE=BOTH;

ALTER SYSTEM SET LOG_ARCHIVE_FORMAT='%t_%s_%r.arc' SCOPE=SPFILE;
ALTER SYSTEM SET LOG_ARCHIVE_MAX_PROCESSES=4 SCOPE=BOTH;

ALTER SYSTEM SET DB_FILE_NAME_CONVERT='/data/oradata/ByteWorks_standby','/data/oradata/ByteWorks' SCOPE=SPFILE;
ALTER SYSTEM SET LOG_FILE_NAME_CONVERT='/data/oradata/ByteWorks_standby','/data/oradata/ByteWorks' SCOPE=SPFILE;

ALTER SYSTEM SET STANDBY_FILE_MANAGEMENT=AUTO SCOPE=BOTH;
ALTER SYSTEM SET FAL_SERVER=BYTEWORK_STANDBY SCOPE=BOTH;
ALTER SYSTEM SET FAL_CLIENT=BYTEWORK_PRIMARY SCOPE=BOTH;

ALTER SYSTEM SET REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE SCOPE=SPFILE;
```

---

## STANDBY

```sql
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(ByteWorks,ByteWorks_standby)' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1='LOCATION=/data/archivelog/ByteWorks VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=ByteWorks_standby' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=BYTEWORK_PRIMARY ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=ByteWorks' SCOPE=BOTH;

ALTER SYSTEM SET LOG_ARCHIVE_FORMAT='%t_%s_%r.arc' SCOPE=SPFILE;
ALTER SYSTEM SET LOG_ARCHIVE_MAX_PROCESSES=4 SCOPE=BOTH;

ALTER SYSTEM SET DB_FILE_NAME_CONVERT='/data/oradata/ByteWorks','/data/oradata/ByteWorks_standby' SCOPE=SPFILE;
ALTER SYSTEM SET LOG_FILE_NAME_CONVERT='/data/oradata/ByteWorks','/data/oradata/ByteWorks_standby' SCOPE=SPFILE;

ALTER SYSTEM SET STANDBY_FILE_MANAGEMENT=AUTO SCOPE=BOTH;
ALTER SYSTEM SET FAL_SERVER=BYTEWORK_PRIMARY SCOPE=BOTH;
ALTER SYSTEM SET FAL_CLIENT=BYTEWORK_STANDBY SCOPE=BOTH;

ALTER SYSTEM SET REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE SCOPE=SPFILE;
```

---

# =========================================
# 5. CREATE STANDBY REDO LOG (SRL)
# =========================================

## Check redo log (PRIMARY)

```sql
SELECT GROUP#, BYTES/1024/1024 AS SIZE_MB FROM V$LOG;
```

---

## Create SRL (PRIMARY)

```sql
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 '/data/oradata/ByteWorks/srl04.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 '/data/oradata/ByteWorks/srl05.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 '/data/oradata/ByteWorks/srl06.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 '/data/oradata/ByteWorks/srl07.log' SIZE 200M;
```

---

## Create SRL (STANDBY)

```sql
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 '/data/oradata/ByteWorks_standby/srl04.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 '/data/oradata/ByteWorks_standby/srl05.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 '/data/oradata/ByteWorks_standby/srl06.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 '/data/oradata/ByteWorks_standby/srl07.log' SIZE 200M;
```

---

## Verify SRL

```sql
SELECT GROUP#, STATUS, BYTES/1024/1024 AS SIZE_MB FROM V$STANDBY_LOG;
```

---

# =========================================
# 6. PASSWORD FILE
# =========================================

```bash
orapwd file=$ORACLE_HOME/dbs/orapwBYTEWORK password=YourPassword force=y
scp $ORACLE_HOME/dbs/orapwBYTEWORK oracle@STANDBY_IP:$ORACLE_HOME/dbs/
```

---

# =========================================
# 7. CREATE STANDBY DATABASE
# =========================================

```rman
recover standby database from service BYTEWORK_PRIMARY;
```

---

# =========================================
# 8. OPEN STANDBY
# =========================================

```sql
ALTER DATABASE OPEN READ ONLY;
```

---

# =========================================
# 9. ENABLE MRP (REAL-TIME APPLY)
# =========================================

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT USING CURRENT LOGFILE;
```

---

# =========================================
# 10. VALIDATION
# =========================================

```sql
SELECT NAME, OPEN_MODE, DATABASE_ROLE FROM V$DATABASE;
SELECT PROCESS, STATUS, THREAD#, SEQUENCE# FROM V$MANAGED_STANDBY;
```

---

# =========================================
# 11. CHECK GAP
# =========================================

```sql
ALTER SESSION SET nls_date_format='DD-MM-YYYY HH24:MI:SS';

SELECT A.THREAD#, B.LAST_SEQ, A.APPLIED_SEQ,
       A.LAST_APP_TIMESTAMP,
       B.LAST_SEQ-A.APPLIED_SEQ ARC_DIFF
FROM (SELECT THREAD#, MAX(SEQUENCE#) APPLIED_SEQ, MAX(NEXT_TIME) LAST_APP_TIMESTAMP
      FROM GV$ARCHIVED_LOG
      WHERE APPLIED = 'YES' OR APPLIED='IN-MEMORY'
      GROUP BY THREAD#) A,
     (SELECT THREAD#, MAX(SEQUENCE#) LAST_SEQ
      FROM GV$ARCHIVED_LOG
      GROUP BY THREAD#) B
WHERE A.THREAD#=B.THREAD#;
```

---

# =========================================
# 12. TEST LOG SHIPPING
# =========================================

```sql
ALTER SYSTEM ARCHIVE LOG CURRENT;
```

---

# =========================================
# 13. ENABLE BROKER (OPTIONAL)
# =========================================

```sql
ALTER SYSTEM SET DG_BROKER_START=TRUE SCOPE=BOTH;
```

---

# =========================================
# 14. RESTART (REQUIRED)
# =========================================

```sql
SHUTDOWN IMMEDIATE;
STARTUP;
```
