# Oracle Database Housekeeping

## Tujuan Pembelajaran

Setelah menyelesaikan materi ini, peserta mampu:

- Memahami konsep housekeeping Oracle Database.
- Mengidentifikasi filesystem yang mulai penuh.
- Melakukan housekeeping Audit File.
- Melakukan housekeeping Trace File.
- Melakukan housekeeping Alert Log.
- Melakukan housekeeping Listener Log.
- Melakukan housekeeping Archive Log menggunakan RMAN.
- Melakukan verifikasi setelah housekeeping.
- Memahami best practice housekeeping di lingkungan Production.

---

# Daftar Isi

1. Apa itu Housekeeping
2. Housekeeping Audit File
3. Housekeeping Trace File
4. Housekeeping Alert Log
5. Housekeeping Listener Log
6. Housekeeping Archive Log
7. Best Practice
8. Ringkasan

---

# 1. Apa itu Housekeeping

Housekeeping merupakan aktivitas rutin Database Administrator (DBA) untuk membersihkan file yang sudah tidak diperlukan agar penggunaan storage tetap terjaga dan database tetap berjalan dengan optimal.

File yang biasanya dilakukan housekeeping:

- Audit File (.aud)
- Trace File (.trc)
- Trace Metadata (.trm)
- Alert Log
- Listener Log
- Archive Log

---

# 2. Housekeeping Audit File

## Studi Kasus

Monitoring menunjukkan filesystem `/u01` sudah mencapai 88%.

Cek penggunaan filesystem.

```bash
df -h
```

Contoh output

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       300G  264G   36G   88% /u01
```

Lakukan pengecekan lokasi Audit File.

```sql
SHOW PARAMETER audit_file_dest;
```

Contoh output

```
/u01/app/oracle/admin/ORCL/adump
```

Masuk ke direktori Audit.

```bash
cd /u01/app/oracle/admin/ORCL/adump
```

Lihat ukuran direktori.

```bash
du -sh .
```

Lihat file yang ada.

```bash
ls -lh
```

Cari file audit yang berumur lebih dari 30 hari.

```bash
find . -name "*.aud" -mtime +30
```

Hapus file.

```bash
find . -name "*.aud" -mtime +30 -delete
```

Verifikasi.

```bash
df -h
```

---

# 3. Housekeeping Trace File

## Studi Kasus

Filesystem `/u01` mencapai 90%.

```bash
df -h
```

Cari lokasi Trace File.

```sql
SELECT VALUE
FROM V$DIAG_INFO
WHERE NAME='Diag Trace';
```

Masuk ke direktori.

```bash
cd /u01/app/oracle/diag/rdbms/ORCL/orcl/trace
```

Lihat ukuran.

```bash
du -sh .
```

Lihat file terbesar.

```bash
ls -lhS
```

Cari Trace File lebih dari 30 hari.

```bash
find . -name "*.trc" -mtime +30
```

Cari Metadata Trace.

```bash
find . -name "*.trm" -mtime +30
```

Hapus Trace File.

```bash
find . -name "*.trc" -mtime +30 -delete
```

Hapus Metadata Trace.

```bash
find . -name "*.trm" -mtime +30 -delete
```

Verifikasi.

```bash
df -h
```

---

# 4. Housekeeping Alert Log

## Studi Kasus

Alert Log mencapai ukuran beberapa GB.

Cari lokasi Alert Log.

```sql
SELECT VALUE
FROM V$DIAG_INFO
WHERE NAME='Diag Trace';
```

Masuk ke direktori.

```bash
cd /u01/app/oracle/diag/rdbms/ORCL/orcl/trace
```

Cari Alert Log.

```bash
ls -lh alert_*.log
```

Lihat ukuran.

```bash
du -sh alert_*.log
```

Backup terlebih dahulu jika diperlukan.

```bash
cp alert_orcl.log /backup/
```

Kosongkan file Alert Log (truncate tanpa menghapus file).

```bash
> alert_orcl.log
```

Verifikasi.

```bash
ls -lh alert_*.log
```

---

# 5. Housekeeping Listener Log

## Studi Kasus

Listener Log terus bertambah hingga beberapa GB.

Cari lokasi Listener Log.

```bash
lsnrctl status
```

Masuk ke direktori log listener.

Contoh

```bash
cd $ORACLE_BASE/diag/tnslsnr/$(hostname)/listener/trace
```

Lihat ukuran.

```bash
du -sh listener.log
```

Backup.

```bash
cp listener.log /backup/
```

Kosongkan file.

```bash
> listener.log
```

Verifikasi.

```bash
ls -lh listener.log
```

---

# 6. Housekeeping Archive Log

## Studi Kasus

Filesystem `/archive` mencapai 91%.

```bash
df -h
```

Contoh output

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       500G  456G   44G   91% /archive
```

Jangan langsung menghapus Archive Log.

Lakukan investigasi terlebih dahulu.

---

## Langkah 1 - Cek Archive Log

```sql
ARCHIVE LOG LIST;
```

Melihat daftar Archive Log.

```sql
SELECT
    THREAD#,
    SEQUENCE#,
    FIRST_TIME,
    APPLIED
FROM V$ARCHIVED_LOG
ORDER BY SEQUENCE# DESC;
```

---

## Langkah 2 - Jika Menggunakan Oracle Data Guard

Pastikan Standby tidak tertinggal.

```sql
ALTER SESSION SET NLS_DATE_FORMAT='DD-MM-YYYY HH24:MI:SS';
```

```sql
SELECT
    A.THREAD#,
    B.LAST_SEQ,
    A.APPLIED_SEQ,
    A.LAST_APP_TIMESTAMP,
    B.LAST_SEQ-A.APPLIED_SEQ ARC_DIFF
FROM
(
    SELECT
        THREAD#,
        MAX(SEQUENCE#) APPLIED_SEQ,
        MAX(NEXT_TIME) LAST_APP_TIMESTAMP
    FROM GV$ARCHIVED_LOG
    WHERE APPLIED IN ('YES','IN-MEMORY')
    GROUP BY THREAD#
) A,
(
    SELECT
        THREAD#,
        MAX(SEQUENCE#) LAST_SEQ
    FROM GV$ARCHIVED_LOG
    GROUP BY THREAD#
) B
WHERE A.THREAD#=B.THREAD#;
```

Pastikan ARC_DIFF kecil atau 0.

---

## Langkah 3 - Pastikan Archive Log Sudah Dibackup

Masuk RMAN.

```bash
rman target /
```

```rman
LIST BACKUP OF ARCHIVELOG ALL;
```

---

## Langkah 4 - Preview Archive Log

Berdasarkan tanggal.

```rman
LIST ARCHIVELOG UNTIL TIME 'SYSDATE-7';
```

---

## Langkah 5 - Delete Berdasarkan SYSDATE

```rman
DELETE NOPROMPT ARCHIVELOG UNTIL TIME 'SYSDATE-7';
```

---

## Langkah 6 - Delete Berdasarkan Sequence

Misalnya sampai sequence 12000.

```rman
DELETE NOPROMPT ARCHIVELOG UNTIL SEQUENCE 12000;
```

---

## Langkah 7 - Crosscheck

```rman
CROSSCHECK ARCHIVELOG ALL;
```

---

## Langkah 8 - Delete Expired

```rman
DELETE EXPIRED ARCHIVELOG ALL;
```

---

## Langkah 9 - Verifikasi

```rman
LIST ARCHIVELOG ALL;
```

Cek kembali filesystem.

```bash
df -h
```

Contoh output

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       500G  298G  202G   60% /archive
```

Housekeeping berhasil dilakukan.

---

# 7. Best Practice

- Jangan menghapus Archive Log menggunakan perintah `rm`.
- Selalu gunakan RMAN untuk housekeeping Archive Log.
- Pastikan Archive Log sudah dibackup sebelum dihapus.
- Pastikan Archive Log sudah diterapkan ke Standby Database.
- Audit File, Trace File, Alert Log, dan Listener Log dapat dibersihkan dari level Operating System.
- Backup file penting sebelum melakukan housekeeping.
- Lakukan housekeeping secara berkala menggunakan Cron atau Scheduler.
- Selalu lakukan verifikasi setelah housekeeping selesai.

---

# 8. Ringkasan

| Housekeeping | Cara Cek | Cara Housekeeping |
|--------------|----------|-------------------|
| Audit File | SHOW PARAMETER audit_file_dest | find *.aud -mtime +30 -delete |
| Trace File | V$DIAG_INFO | find *.trc / *.trm -mtime +30 -delete |
| Alert Log | V$DIAG_INFO | Backup lalu truncate file |
| Listener Log | lsnrctl status | Backup lalu truncate file |
| Archive Log | ARCHIVE LOG LIST | RMAN DELETE ARCHIVELOG |
| Archive by Date | LIST ARCHIVELOG UNTIL TIME | DELETE ARCHIVELOG UNTIL TIME |
| Archive by Sequence | V$ARCHIVED_LOG | DELETE ARCHIVELOG UNTIL SEQUENCE |
| Crosscheck | RMAN | CROSSCHECK ARCHIVELOG ALL |
| Delete Expired | RMAN | DELETE EXPIRED ARCHIVELOG ALL |
