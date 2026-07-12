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

# 11. Monitoring Replication (Oracle Data Guard)

Tujuan

Memastikan proses replikasi dari Primary Database ke Standby Database berjalan dengan normal.

## Cek Database Role

```sql
SELECT
    NAME,
    DATABASE_ROLE,
    OPEN_MODE
FROM V$DATABASE;
```

Contoh Output

| NAME | DATABASE_ROLE | OPEN_MODE |
|------|---------------|-----------|
| ORCL | PRIMARY | READ WRITE |

atau

| NAME | DATABASE_ROLE | OPEN_MODE |
|------|---------------|-----------|
| ORCL | PHYSICAL STANDBY | READ ONLY WITH APPLY |

---

## Cek Managed Recovery Process (Standby)

```sql
SELECT
    PROCESS,
    STATUS,
    THREAD#,
    SEQUENCE#
FROM V$MANAGED_STANDBY;
```

Yang perlu diperhatikan

- Pastikan proses **MRP0** berjalan.
- Pastikan proses **RFS** menerima redo dari Primary Database.

---

## Cek Archive Log yang Sudah Diterapkan

```sql
SELECT
    THREAD#,
    SEQUENCE#,
    APPLIED
FROM V$ARCHIVED_LOG
ORDER BY SEQUENCE# DESC;
```

Yang perlu diperhatikan

- Nilai **APPLIED = YES** menandakan archive log telah berhasil diterapkan ke Standby Database.

---

## Jika Menggunakan Data Guard Broker

```sql
DGMGRL> SHOW CONFIGURATION;
```

Yang perlu diperhatikan

- Status konfigurasi harus **SUCCESS**.
- Tidak terdapat pesan **WARNING** atau **ERROR**.

---

# 12. Monitoring Top 10 SQL

Tujuan

Mengetahui SQL yang paling banyak menggunakan resource sehingga dapat menjadi kandidat untuk analisis atau optimasi.

## Top 10 SQL Berdasarkan CPU Time

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

Yang perlu diperhatikan

- SQL dengan penggunaan CPU tinggi.
- SQL yang dieksekusi berulang kali.
- SQL dengan waktu eksekusi yang lama.

---

## Top 10 SQL Berdasarkan Elapsed Time

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

Yang perlu diperhatikan

- Query yang membutuhkan waktu paling lama untuk selesai.
- Tidak selalu disebabkan oleh CPU; bisa juga karena I/O, locking, atau menunggu resource lain.

---

## Melihat Detail SQL

```sql
SELECT
    SQL_ID,
    SQL_TEXT
FROM V$SQL
WHERE SQL_ID = '&SQL_ID';
```

Masukkan nilai `SQL_ID` yang ingin diperiksa untuk melihat teks SQL secara lengkap atau sebagian.

---

# Kapan DBA Harus Melakukan Analisis?

Lakukan investigasi lebih lanjut apabila:

- SQL menggunakan CPU sangat tinggi.
- SQL memiliki waktu eksekusi yang lama.
- SQL dieksekusi ribuan kali dalam waktu singkat.
- Replikasi Data Guard mengalami keterlambatan (lag).
- Archive Log pada Standby belum berstatus **APPLIED**.
- Status Data Guard Broker bukan **SUCCESS**.


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
