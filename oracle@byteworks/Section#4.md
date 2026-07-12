# Oracle Database Monitoring

Monitoring adalah aktivitas yang dilakukan oleh Database Administrator (DBA) untuk memastikan database berjalan dengan baik, memiliki kapasitas penyimpanan yang cukup, serta dapat digunakan oleh aplikasi tanpa kendala.

---

# Tujuan Pembelajaran

Setelah menyelesaikan materi ini, peserta mampu:

- Memahami konsep monitoring Oracle Database.
- Melakukan pengecekan kesehatan database.
- Memastikan kapasitas storage masih mencukupi.
- Melihat user yang sedang menggunakan database.
- Mengidentifikasi blocking session.
- Memantau status Oracle Data Guard.
- Mengetahui SQL yang menggunakan resource terbesar.
- Menentukan tindakan awal ketika terjadi masalah.

---

# Daftar Isi

1. Monitoring Status Database
2. Monitoring Instance
3. Monitoring Listener
4. Monitoring Tablespace
5. Monitoring Free Space
6. Monitoring Datafile
7. Monitoring Session
8. Monitoring Blocking Session
9. Monitoring Locked Object
10. Monitoring Long Running Query
11. Monitoring Archive Log
12. Monitoring Replication (Oracle Data Guard)
13. Monitoring Top 10 SQL
14. Monitoring Alert Log
15. Monitoring Invalid Object
16. Monitoring Database Version
17. Daily Health Check
18. Studi Kasus
19. Best Practice

---

# 1. Monitoring Status Database

## Tujuan

Memastikan database dalam keadaan normal.

```sql
SELECT
    NAME,
    OPEN_MODE,
    DATABASE_ROLE
FROM V$DATABASE;
```

### Yang perlu diperhatikan

- OPEN_MODE = READ WRITE
- DATABASE_ROLE = PRIMARY atau PHYSICAL STANDBY

---

# 2. Monitoring Instance

## Tujuan

Melihat kondisi Instance Oracle.

```sql
SELECT
    INSTANCE_NAME,
    STATUS,
    HOST_NAME,
    STARTUP_TIME
FROM V$INSTANCE;
```

### Yang perlu diperhatikan

- STATUS = OPEN
- HOST_NAME
- STARTUP_TIME

---

# 3. Monitoring Listener

## Tujuan

Memastikan Listener Oracle berjalan.

Linux / Windows

```bash
lsnrctl status
```

### Yang perlu diperhatikan

- Listener Status
- Registered Services
- Listening Endpoint

---

# 4. Monitoring Tablespace

## Tujuan

Melihat ukuran setiap tablespace.

```sql
SELECT
    TABLESPACE_NAME,
    ROUND(SUM(BYTES)/1024/1024,2) SIZE_MB
FROM DBA_DATA_FILES
GROUP BY TABLESPACE_NAME;
```

### Yang perlu diperhatikan

- Ukuran setiap tablespace
- Pastikan masih tersedia ruang yang cukup

---

# 5. Monitoring Free Space

## Tujuan

Melihat sisa ruang pada setiap tablespace.

```sql
SELECT
    TABLESPACE_NAME,
    ROUND(SUM(BYTES)/1024/1024,2) FREE_MB
FROM DBA_FREE_SPACE
GROUP BY TABLESPACE_NAME;
```

### Yang perlu diperhatikan

- Free space tidak boleh habis
- Segera lakukan penambahan datafile jika diperlukan

---

# 6. Monitoring Datafile

## Tujuan

Melihat datafile yang digunakan database.

```sql
SELECT
    FILE_NAME,
    TABLESPACE_NAME,
    AUTOEXTENSIBLE
FROM DBA_DATA_FILES;
```

### Yang perlu diperhatikan

- Lokasi Datafile
- Status Autoextend

---

# 7. Monitoring Session

## Tujuan

Melihat user yang sedang menggunakan database.

```sql
SELECT
    SID,
    SERIAL#,
    USERNAME,
    STATUS,
    MACHINE,
    PROGRAM
FROM V$SESSION
WHERE USERNAME IS NOT NULL;
```

### Yang perlu diperhatikan

- ACTIVE
- INACTIVE
- User yang sedang login

---

# 8. Monitoring Blocking Session

## Tujuan

Mengetahui apakah terdapat session yang menghambat session lain.

```sql
SELECT
    SID,
    SERIAL#,
    USERNAME,
    BLOCKING_SESSION
FROM V$SESSION
WHERE BLOCKING_SESSION IS NOT NULL;
```

### Yang perlu diperhatikan

- BLOCKING_SESSION tidak boleh bernilai NULL jika terjadi blocking
- Lakukan investigasi sebelum melakukan KILL SESSION

---

# 9. Monitoring Locked Object

## Tujuan

Mengetahui object yang sedang dikunci.

```sql
SELECT
    LO.SESSION_ID,
    DO.OBJECT_NAME,
    DO.OBJECT_TYPE,
    LO.LOCKED_MODE
FROM V$LOCKED_OBJECT LO
JOIN DBA_OBJECTS DO
ON LO.OBJECT_ID = DO.OBJECT_ID;
```

---

# 10. Monitoring Long Running Query

## Tujuan

Mengetahui query yang masih berjalan dalam waktu lama.

```sql
SELECT
    SID,
    SERIAL#,
    OPNAME,
    SOFAR,
    TOTALWORK
FROM V$SESSION_LONGOPS
WHERE SOFAR <> TOTALWORK;
```

---

# 11. Monitoring Archive Log

## Tujuan

Memastikan database berjalan dalam mode ARCHIVELOG.

```sql
SELECT
    LOG_MODE
FROM V$DATABASE;
```

Melihat informasi Archive Log.

```sql
ARCHIVE LOG LIST;
```

---

# 12. Monitoring Replication (Oracle Data Guard)

## Tujuan

Memastikan proses replikasi antara Primary Database dan Standby Database berjalan dengan normal.

### Cek Role Database

```sql
SELECT
    NAME,
    DATABASE_ROLE,
    OPEN_MODE
FROM V$DATABASE;
```

### Cek Managed Recovery Process

```sql
SELECT
    PROCESS,
    STATUS,
    THREAD#,
    SEQUENCE#
FROM V$MANAGED_STANDBY;
```

Yang perlu diperhatikan:

- MRP0 berjalan.
- RFS menerima redo dari Primary.

### Check Gap Sequence

Sebelum menjalankan query:

```sql
ALTER SESSION SET NLS_DATE_FORMAT='DD-MM-YYYY HH24:MI:SS';
```

Jalankan query berikut:

```sql
SELECT
    A.THREAD#,
    B.LAST_SEQ,
    A.APPLIED_SEQ,
    A.LAST_APP_TIMESTAMP,
    B.LAST_SEQ - A.APPLIED_SEQ AS ARC_DIFF
FROM
(
    SELECT
        THREAD#,
        MAX(SEQUENCE#) AS APPLIED_SEQ,
        MAX(NEXT_TIME) AS LAST_APP_TIMESTAMP
    FROM GV$ARCHIVED_LOG
    WHERE APPLIED IN ('YES','IN-MEMORY')
    GROUP BY THREAD#
) A,
(
    SELECT
        THREAD#,
        MAX(SEQUENCE#) AS LAST_SEQ
    FROM GV$ARCHIVED_LOG
    GROUP BY THREAD#
) B
WHERE A.THREAD# = B.THREAD#;
```

### Interpretasi

| ARC_DIFF | Keterangan |
|-----------|------------|
| 0 | Replikasi normal |
| 1 - 5 | Standby sedang mengejar archive log |
| > 5 | Perlu dilakukan investigasi |

### Jika Menggunakan Data Guard Broker

```bash
dgmgrl

SHOW CONFIGURATION;
```

Pastikan status:

```
SUCCESS
```

---

# 13. Monitoring Top 10 SQL

## Tujuan

Mengetahui SQL yang paling banyak menggunakan resource.

### Top SQL Berdasarkan CPU

```sql
SELECT
    SQL_ID,
    EXECUTIONS,
    CPU_TIME/1000000 CPU_SECONDS,
    ELAPSED_TIME/1000000 ELAPSED_SECONDS
FROM V$SQL
ORDER BY CPU_TIME DESC
FETCH FIRST 10 ROWS ONLY;
```

### Top SQL Berdasarkan Elapsed Time

```sql
SELECT
    SQL_ID,
    EXECUTIONS,
    ELAPSED_TIME/1000000 ELAPSED_SECONDS,
    CPU_TIME/1000000 CPU_SECONDS
FROM V$SQL
ORDER BY ELAPSED_TIME DESC
FETCH FIRST 10 ROWS ONLY;
```

### Melihat Isi SQL

```sql
SELECT
    SQL_ID,
    SQL_TEXT
FROM V$SQL
WHERE SQL_ID='&SQL_ID';
```

---

# 14. Monitoring Alert Log

## Tujuan

Mengetahui lokasi Alert Log Oracle.

```sql
SELECT
    VALUE
FROM V$DIAG_INFO
WHERE NAME='Diag Trace';
```

Kemudian buka file:

```
alert_<SID>.log
```

Yang perlu diperhatikan:

- ORA- Error
- Startup Database
- Shutdown Database
- Crash Database

---

# 15. Monitoring Invalid Object

## Tujuan

Mengetahui object yang gagal dikompilasi.

```sql
SELECT
    OWNER,
    OBJECT_NAME,
    OBJECT_TYPE
FROM DBA_OBJECTS
WHERE STATUS='INVALID';
```

---

# 16. Monitoring Database Version

## Tujuan

Mengetahui versi Oracle Database.

```sql
SELECT
    BANNER
FROM V$VERSION;
```

---

# 17. Daily Health Check

Checklist yang dapat dilakukan setiap hari:

- Cek Status Database
- Cek Status Instance
- Cek Listener
- Cek Tablespace
- Cek Free Space
- Cek Datafile
- Cek Session
- Cek Blocking Session
- Cek Long Running Query
- Cek Archive Log
- Cek Status Data Guard
- Cek Gap Sequence
- Cek Top 10 SQL
- Cek Alert Log
- Cek Invalid Object

---

# 18. Studi Kasus

## Kasus 1

Aplikasi tidak dapat login.

Langkah pemeriksaan:

- Status Database
- Instance
- Listener

---

## Kasus 2

Aplikasi berjalan lambat.

Langkah pemeriksaan:

- Session
- Blocking Session
- Long Running Query
- Top SQL

---

## Kasus 3

Standby Database tertinggal.

Langkah pemeriksaan:

- Data Guard Broker
- Managed Recovery Process
- Gap Sequence
- Archive Log

---

## Kasus 4

Developer tidak dapat membuat table.

Langkah pemeriksaan:

- User Privilege
- Tablespace
- User Quota

---

# 19. Best Practice

- Lakukan monitoring database secara rutin setiap hari.
- Jangan langsung melakukan `KILL SESSION` tanpa mengetahui penyebabnya.
- Pastikan tablespace memiliki ruang kosong yang cukup.
- Aktifkan Autoextend jika memang diperlukan.
- Periksa Alert Log secara berkala.
- Monitor replikasi Oracle Data Guard setiap hari.
- Dokumentasikan setiap masalah dan tindakan yang dilakukan.
- Gunakan dashboard monitoring seperti Grafana atau Oracle Enterprise Manager untuk mempermudah pemantauan.

---

# Ringkasan

| Monitoring | View / Command |
|------------|----------------|
| Database Status | V$DATABASE |
| Instance | V$INSTANCE |
| Listener | lsnrctl status |
| Tablespace | DBA_DATA_FILES |
| Free Space | DBA_FREE_SPACE |
| Datafile | DBA_DATA_FILES |
| Session | V$SESSION |
| Blocking Session | V$SESSION |
| Locked Object | V$LOCKED_OBJECT |
| Long Running Query | V$SESSION_LONGOPS |
| Archive Log | V$DATABASE, ARCHIVE LOG LIST |
| Data Guard | V$MANAGED_STANDBY, GV$ARCHIVED_LOG, DGMGRL |
| Top SQL | V$SQL |
| Alert Log | V$DIAG_INFO |
| Invalid Object | DBA_OBJECTS |
| Database Version | V$VERSION |
