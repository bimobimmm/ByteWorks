# Oracle Database Monitoring

Monitoring merupakan aktivitas penting seorang Database Administrator (DBA) untuk memastikan database berjalan dengan optimal, stabil, dan aman. Oracle menyediakan berbagai Dynamic Performance Views (`V$`) dan Data Dictionary Views (`DBA_`) untuk memantau kondisi database secara real-time.

---

# Daftar Isi

- Database Status
- Instance Information
- Database Version
- Session Monitoring
- Active Session
- Blocking Session
- Locked Object
- Long Running Query
- Tablespace Usage
- Datafile Monitoring
- Temp Tablespace
- Undo Tablespace
- Archive Log
- FRA Usage
- SGA Monitoring
- PGA Monitoring
- CPU Usage
- Memory Usage
- Wait Event
- Invalid Object
- Alert Log
- Best Practice

---

# 1. Database Status

Mengetahui status database.

```sql
SELECT name,
       open_mode,
       database_role
FROM v$database;
```

---

# 2. Instance Information

```sql
SELECT instance_name,
       host_name,
       version,
       status,
       startup_time
FROM v$instance;
```

---

# 3. Database Version

```sql
SELECT banner
FROM v$version;
```

---

# 4. Session Monitoring

Melihat seluruh session yang sedang aktif.

```sql
SELECT sid,
       serial#,
       username,
       status,
       machine,
       program
FROM v$session
ORDER BY username;
```

---

# 5. Active Session

```sql
SELECT sid,
       serial#,
       username,
       sql_id,
       event
FROM v$session
WHERE status='ACTIVE';
```

---

# 6. Blocking Session

Mengetahui session yang menyebabkan blocking.

```sql
SELECT sid,
       serial#,
       username,
       blocking_session
FROM v$session
WHERE blocking_session IS NOT NULL;
```

---

# 7. Locked Object

Melihat object yang sedang terkunci.

```sql
SELECT
    lo.session_id,
    do.object_name,
    do.object_type,
    lo.locked_mode
FROM v$locked_object lo
JOIN dba_objects do
ON lo.object_id = do.object_id;
```

---

# 8. Long Running Query

```sql
SELECT sid,
       serial#,
       opname,
       sofar,
       totalwork,
       elapsed_seconds
FROM v$session_longops
WHERE sofar <> totalwork;
```

---

# 9. Tablespace Usage

```sql
SELECT
    tablespace_name,
    ROUND(SUM(bytes)/1024/1024,2) size_mb
FROM dba_data_files
GROUP BY tablespace_name;
```

---

# 10. Free Space

```sql
SELECT
    tablespace_name,
    ROUND(SUM(bytes)/1024/1024,2) free_mb
FROM dba_free_space
GROUP BY tablespace_name;
```

---

# 11. Datafile Monitoring

```sql
SELECT
    file_name,
    tablespace_name,
    autoextensible
FROM dba_data_files;
```

---

# 12. Temporary Tablespace

```sql
SELECT
    tablespace_name,
    SUM(bytes_used)/1024/1024 used_mb,
    SUM(bytes_free)/1024/1024 free_mb
FROM v$temp_space_header
GROUP BY tablespace_name;
```

---

# 13. Undo Tablespace

```sql
SELECT
    tablespace_name,
    status,
    SUM(bytes)/1024/1024 size_mb
FROM dba_undo_extents
GROUP BY tablespace_name,status;
```

---

# 14. Archive Log

Archive Log Mode.

```sql
SELECT log_mode
FROM v$database;
```

Archive Log Destination.

```sql
ARCHIVE LOG LIST;
```

---

# 15. Fast Recovery Area (FRA)

```sql
SELECT
    name,
    space_limit/1024/1024 size_mb,
    space_used/1024/1024 used_mb
FROM v$recovery_file_dest;
```

---

# 16. SGA Monitoring

```sql
SHOW SGA;
```

atau

```sql
SELECT *
FROM v$sga;
```

---

# 17. PGA Monitoring

```sql
SELECT
    name,
    value
FROM v$pgastat;
```

---

# 18. CPU Usage

```sql
SELECT
    stat_name,
    value
FROM v$osstat
WHERE stat_name LIKE '%CPU%';
```

---

# 19. Memory Usage

```sql
SELECT
    component,
    current_size/1024/1024 current_mb
FROM v$memory_dynamic_components;
```

---

# 20. Wait Event

```sql
SELECT
    event,
    total_waits,
    time_waited
FROM v$system_event
ORDER BY total_waits DESC;
```

---

# 21. Invalid Object

```sql
SELECT owner,
       object_name,
       object_type
FROM dba_objects
WHERE status='INVALID';
```

---

# 22. Alert Log Location

```sql
SELECT value
FROM v$diag_info
WHERE name='Diag Trace';
```

---

# 23. Killing Session

```sql
ALTER SYSTEM KILL SESSION 'SID,SERIAL#';
```

Contoh:

```sql
ALTER SYSTEM KILL SESSION '123,456';
```

---

# 24. Monitoring Top SQL

```sql
SELECT
    sql_id,
    executions,
    elapsed_time,
    cpu_time
FROM v$sql
ORDER BY elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;
```

---

# 25. Best Practice

- Monitor session aktif secara berkala.
- Periksa blocking session sebelum melakukan kill session.
- Pastikan penggunaan tablespace tidak melebihi 85%.
- Pantau penggunaan TEMP dan UNDO.
- Periksa FRA agar tidak penuh.
- Monitor wait event untuk mendeteksi bottleneck.
- Cek invalid object setelah deployment aplikasi.
- Simpan hasil monitoring sebagai baseline performa.

---

# Ringkasan

| Monitoring | View |
|------------|------|
| Database | V$DATABASE |
| Instance | V$INSTANCE |
| Version | V$VERSION |
| Session | V$SESSION |
| Blocking | V$SESSION |
| Lock | V$LOCKED_OBJECT |
| Long Operation | V$SESSION_LONGOPS |
| Tablespace | DBA_DATA_FILES |
| Free Space | DBA_FREE_SPACE |
| Temp | V$TEMP_SPACE_HEADER |
| Undo | DBA_UNDO_EXTENTS |
| Archive Log | V$DATABASE |
| FRA | V$RECOVERY_FILE_DEST |
| SGA | V$SGA |
| PGA | V$PGASTAT |
| CPU | V$OSSTAT |
| Wait Event | V$SYSTEM_EVENT |
| Invalid Object | DBA_OBJECTS |
| Top SQL | V$SQL |

---

# Latihan

1. Cek status database dan instance.
2. Lihat seluruh session yang sedang aktif.
3. Identifikasi apakah ada blocking session.
4. Cek penggunaan tablespace dan free space.
5. Periksa penggunaan TEMP dan UNDO.
6. Pastikan FRA tidak melebihi 80%.
7. Lihat Top 10 SQL berdasarkan elapsed time.
8. Cek apakah terdapat invalid object.
9. Temukan lokasi Alert Log.
10. Kill salah satu session (gunakan session uji coba).

---

**Author:** Bimo Anggoro Jati
