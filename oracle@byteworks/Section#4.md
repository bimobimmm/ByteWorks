# Database Monitoring

Database Monitoring adalah aktivitas yang dilakukan Database Administrator (DBA) untuk memastikan database berjalan dengan baik, memiliki kapasitas penyimpanan yang cukup, dan dapat digunakan oleh aplikasi tanpa kendala.

---

# Tujuan Pembelajaran

Setelah menyelesaikan materi ini, peserta mampu:

- Memahami pentingnya monitoring database.
- Mengecek status database.
- Mengecek status instance.
- Mengecek tablespace.
- Mengecek session yang sedang aktif.
- Mengecek blocking session.
- Mengecek Alert Log.
- Menentukan tindakan awal ketika terjadi masalah.

---

# 1. Monitoring Status Database

Tujuan:

Memastikan database dalam keadaan normal.

Query:

```sql
SELECT
    NAME,
    OPEN_MODE,
    DATABASE_ROLE
FROM V$DATABASE;
```

Contoh Output

| NAME | OPEN_MODE | DATABASE_ROLE |
|------|-----------|---------------|
| ORCL | READ WRITE | PRIMARY |

Yang perlu diperhatikan:

- Database harus berada pada **READ WRITE**
- Pastikan role sesuai (PRIMARY atau PHYSICAL STANDBY)

---

# 2. Monitoring Status Instance

Tujuan:

Melihat kondisi instance Oracle.

Query

```sql
SELECT
    INSTANCE_NAME,
    STATUS,
    HOST_NAME,
    STARTUP_TIME
FROM V$INSTANCE;
```

Yang perlu diperhatikan

- STATUS = OPEN
- Kapan terakhir database di-restart

---

# 3. Monitoring Tablespace

Tujuan

Memastikan ruang penyimpanan database masih mencukupi.

Query

```sql
SELECT
    TABLESPACE_NAME,
    ROUND(SUM(BYTES)/1024/1024,2) SIZE_MB
FROM DBA_DATA_FILES
GROUP BY TABLESPACE_NAME;
```

Yang perlu diperhatikan

- Ukuran setiap tablespace
- Apakah ada tablespace yang ukurannya terlalu kecil

---

# 4. Monitoring Free Space

Tujuan

Melihat sisa ruang yang tersedia.

Query

```sql
SELECT
    TABLESPACE_NAME,
    ROUND(SUM(BYTES)/1024/1024,2) FREE_MB
FROM DBA_FREE_SPACE
GROUP BY TABLESPACE_NAME;
```

Yang perlu diperhatikan

- Jangan sampai free space habis.

---

# 5. Monitoring Session

Tujuan

Melihat user yang sedang menggunakan database.

Query

```sql
SELECT
    SID,
    USERNAME,
    STATUS,
    MACHINE
FROM V$SESSION
WHERE USERNAME IS NOT NULL;
```

Yang perlu diperhatikan

- User yang login
- Status ACTIVE atau INACTIVE

---

# 6. Monitoring Blocking Session

Tujuan

Mengetahui apakah ada session yang menghambat session lain.

Query

```sql
SELECT
    SID,
    SERIAL#,
    USERNAME,
    BLOCKING_SESSION
FROM V$SESSION
WHERE BLOCKING_SESSION IS NOT NULL;
```

Yang perlu diperhatikan

- Jika terdapat BLOCKING_SESSION, lakukan investigasi sebelum mengambil tindakan.

---

# 7. Monitoring Alert Log

Tujuan

Mengetahui lokasi Alert Log Oracle.

Query

```sql
SELECT VALUE
FROM V$DIAG_INFO
WHERE NAME = 'Diag Trace';
```

Kemudian buka file:

```
alert_<SID>.log
```

Yang perlu diperhatikan

- ORA- Error
- Startup database
- Shutdown database
- Crash database

---

# 8. Monitoring Archive Log Mode

Tujuan

Memastikan database berjalan pada mode ARCHIVELOG atau NOARCHIVELOG.

Query

```sql
SELECT LOG_MODE
FROM V$DATABASE;
```

Output

| LOG_MODE |
|----------|
| ARCHIVELOG |

---

# 9. Monitoring Invalid Object

Tujuan

Mengetahui object yang gagal dikompilasi.

Query

```sql
SELECT
    OWNER,
    OBJECT_NAME,
    OBJECT_TYPE
FROM DBA_OBJECTS
WHERE STATUS='INVALID';
```

---

# 10. Monitoring Database Version

Tujuan

Mengetahui versi Oracle Database.

Query

```sql
SELECT BANNER
FROM V$VERSION;
```

---

# Studi Kasus

### Kasus 1

Aplikasi tidak bisa login.

Langkah pertama:

- Cek status database
- Cek instance
- Cek listener

---

### Kasus 2

Aplikasi lambat.

Langkah pertama:

- Cek session aktif
- Cek blocking session
- Cek tablespace

---

### Kasus 3

Developer tidak bisa membuat table.

Langkah pertama:

- Cek status user
- Cek privilege
- Cek quota tablespace

---

# Best Practice

- Lakukan monitoring setiap hari.
- Jangan langsung melakukan kill session tanpa analisis.
- Pastikan tablespace memiliki ruang kosong yang cukup.
- Periksa Alert Log secara berkala.
- Dokumentasikan setiap permasalahan dan tindakan yang dilakukan.

---

# Ringkasan

| Monitoring | Query |
|------------|-------|
| Status Database | V$DATABASE |
| Status Instance | V$INSTANCE |
| Tablespace | DBA_DATA_FILES |
| Free Space | DBA_FREE_SPACE |
| Session | V$SESSION |
| Blocking Session | V$SESSION |
| Alert Log | V$DIAG_INFO |
| Archive Log | V$DATABASE |
| Invalid Object | DBA_OBJECTS |
| Database Version | V$VERSION |
