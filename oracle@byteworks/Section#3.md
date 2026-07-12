# Tablespace Management

Tablespace merupakan logical storage pada Oracle Database yang digunakan untuk menyimpan object seperti table, index, dan segment. Setiap tablespace terdiri dari satu atau lebih datafile yang berada pada sistem operasi.

---

# Daftar Isi

- Apa itu Tablespace
- Jenis Tablespace
- Melihat Tablespace
- Membuat Tablespace
- Menambah Datafile
- Resize Datafile
- Autoextend
- Online / Offline Tablespace
- Read Only / Read Write
- Rename Tablespace
- Drop Tablespace
- Temporary Tablespace
- Default Tablespace
- Monitoring Tablespace
- Best Practice

---

# 1. Apa itu Tablespace

Oracle menggunakan struktur berikut:

```
Database
    │
    ├── Tablespace
    │       │
    │       ├── Datafile
    │       ├── Datafile
    │       └── Datafile
    │
    └── Schema Object
            ├── Table
            ├── Index
            └── LOB
```

Tablespace adalah **logical storage**, sedangkan datafile adalah **physical storage** yang berada di sistem operasi.

---

# 2. Jenis Tablespace

Beberapa tablespace bawaan Oracle:

| Tablespace | Fungsi |
|------------|--------|
| SYSTEM | Menyimpan data dictionary Oracle |
| SYSAUX | Menyimpan komponen tambahan Oracle |
| USERS | Default user tablespace |
| TEMP | Temporary Segment |
| UNDOTBS1 | Undo Segment |

---

# 3. Melihat Semua Tablespace

```sql
SELECT tablespace_name,
       status,
       contents
FROM dba_tablespaces;
```

---

# 4. Melihat Datafile

```sql
SELECT tablespace_name,
       file_name,
       bytes/1024/1024 AS size_mb
FROM dba_data_files;
```

---

# 5. Membuat Tablespace Baru

```sql
CREATE TABLESPACE app_data
DATAFILE '/u01/oradata/ORCL/app_data01.dbf'
SIZE 500M;
```

---

# 6. Membuat Tablespace dengan Autoextend

```sql
CREATE TABLESPACE app_data
DATAFILE '/u01/oradata/ORCL/app_data01.dbf'
SIZE 500M
AUTOEXTEND ON
NEXT 100M
MAXSIZE 5G;
```

---

# 7. Menambah Datafile

```sql
ALTER TABLESPACE app_data
ADD DATAFILE '/u01/oradata/ORCL/app_data02.dbf'
SIZE 1G;
```

---

# 8. Resize Datafile

Melihat file terlebih dahulu.

```sql
SELECT file_name
FROM dba_data_files;
```

Resize.

```sql
ALTER DATABASE
DATAFILE '/u01/oradata/ORCL/app_data01.dbf'
RESIZE 2G;
```

---

# 9. Mengaktifkan Autoextend

```sql
ALTER DATABASE
DATAFILE '/u01/oradata/ORCL/app_data01.dbf'
AUTOEXTEND ON
NEXT 100M
MAXSIZE UNLIMITED;
```

---

# 10. Menonaktifkan Autoextend

```sql
ALTER DATABASE
DATAFILE '/u01/oradata/ORCL/app_data01.dbf'
AUTOEXTEND OFF;
```

---

# 11. Offline Tablespace

```sql
ALTER TABLESPACE app_data
OFFLINE;
```

---

# 12. Online Tablespace

```sql
ALTER TABLESPACE app_data
ONLINE;
```

---

# 13. Read Only Tablespace

```sql
ALTER TABLESPACE app_data
READ ONLY;
```

---

# 14. Read Write Tablespace

```sql
ALTER TABLESPACE app_data
READ WRITE;
```

---

# 15. Rename Tablespace

```sql
ALTER TABLESPACE app_data
RENAME TO application_data;
```

---

# 16. Drop Tablespace

Menghapus tablespace.

```sql
DROP TABLESPACE application_data;
```

Menghapus beserta datafile.

```sql
DROP TABLESPACE application_data
INCLUDING CONTENTS
AND DATAFILES;
```

---

# 17. Temporary Tablespace

Melihat temporary tablespace.

```sql
SELECT tablespace_name
FROM dba_tablespaces
WHERE contents='TEMPORARY';
```

Membuat temporary tablespace.

```sql
CREATE TEMPORARY TABLESPACE temp_app
TEMPFILE '/u01/oradata/ORCL/temp_app01.dbf'
SIZE 1G;
```

---

# 18. Mengubah Default Tablespace User

```sql
ALTER USER trainee
DEFAULT TABLESPACE app_data;
```

Mengubah temporary tablespace.

```sql
ALTER USER trainee
TEMPORARY TABLESPACE temp_app;
```

---

# 19. Monitoring Tablespace

Melihat ukuran tablespace.

```sql
SELECT
    tablespace_name,
    ROUND(SUM(bytes)/1024/1024,2) AS size_mb
FROM dba_data_files
GROUP BY tablespace_name;
```

---

# 20. Monitoring Free Space

```sql
SELECT
    tablespace_name,
    ROUND(SUM(bytes)/1024/1024,2) AS free_mb
FROM dba_free_space
GROUP BY tablespace_name;
```

---

# 21. Monitoring Persentase Penggunaan

```sql
SELECT
    df.tablespace_name,
    ROUND(df.total_mb,2) total_mb,
    ROUND(fs.free_mb,2) free_mb,
    ROUND(df.total_mb-fs.free_mb,2) used_mb,
    ROUND(((df.total_mb-fs.free_mb)/df.total_mb)*100,2) used_percent
FROM
(
    SELECT tablespace_name,
           SUM(bytes)/1024/1024 total_mb
    FROM dba_data_files
    GROUP BY tablespace_name
) df
JOIN
(
    SELECT tablespace_name,
           SUM(bytes)/1024/1024 free_mb
    FROM dba_free_space
    GROUP BY tablespace_name
) fs
ON df.tablespace_name=fs.tablespace_name;
```

---

# 22. Melihat Tablespace Milik User

```sql
SELECT username,
       default_tablespace,
       temporary_tablespace
FROM dba_users;
```

---

# 23. Best Practice

- Pisahkan data aplikasi ke tablespace khusus.
- Jangan menyimpan object aplikasi di SYSTEM atau SYSAUX.
- Aktifkan AUTOEXTEND sesuai kebutuhan.
- Gunakan MAXSIZE agar pertumbuhan tetap terkontrol.
- Monitor penggunaan tablespace secara berkala.
- Tambahkan datafile sebelum tablespace penuh.
- Gunakan TEMP dan UNDO sesuai kapasitas workload.
- Dokumentasikan lokasi seluruh datafile.

---

# Ringkasan

| Perintah | Fungsi |
|----------|--------|
| CREATE TABLESPACE | Membuat tablespace |
| ALTER TABLESPACE ADD DATAFILE | Menambah datafile |
| ALTER DATABASE DATAFILE RESIZE | Mengubah ukuran datafile |
| AUTOEXTEND ON/OFF | Mengatur pertumbuhan otomatis |
| OFFLINE | Menonaktifkan tablespace |
| ONLINE | Mengaktifkan tablespace |
| READ ONLY | Mode hanya baca |
| READ WRITE | Mode baca/tulis |
| DROP TABLESPACE | Menghapus tablespace |
| ALTER USER DEFAULT TABLESPACE | Mengubah default tablespace user |

---

# Latihan

1. Buat tablespace bernama `TRAINING_TS` berukuran 500 MB.
2. Aktifkan AUTOEXTEND dengan penambahan 100 MB hingga maksimal 5 GB.
3. Buat user `training_user` dan gunakan `TRAINING_TS` sebagai default tablespace.
4. Login sebagai `training_user` dan buat tabel `EMPLOYEE`.
5. Tambahkan data ke tabel tersebut.
6. Resize datafile menjadi 1 GB.
7. Tambahkan datafile kedua sebesar 500 MB.
8. Cek total kapasitas dan free space tablespace.
9. Ubah tablespace menjadi `READ ONLY`, lalu kembali ke `READ WRITE`.
10. Hapus tablespace beserta seluruh datafile.

---

**Author:** Bimo Anggoro Jati
