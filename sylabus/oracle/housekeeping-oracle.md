# Oracle Housekeeping — Audit, Log, Trace (ByteWorks)

## Environment

| Parameter | Value |
|----------|------|
| ORACLE_BASE | /data/u01/app/oracle |
| ORACLE_SID | BYTEWORK |
| Audit Directory | $ORACLE_BASE/admin/$ORACLE_SID/adump |
| Trace Directory | $ORACLE_BASE/diag/rdbms/<db_name>/<instance_name>/trace |
| Retention (AUD) | 7 Days |
| Retention (LOG/TRC) | 7 Days |

---

### Audit File Cleanup

#### 1. Navigate to Audit Directory
Masuk ke direktori penyimpanan audit file untuk memastikan lokasi file yang akan dibersihkan.

```bash
cd $ORACLE_BASE/admin/$ORACLE_SID/adump
ls -lh
```

#### 2. Validate Files
Melihat daftar file audit yang tersedia sebagai langkah verifikasi sebelum dilakukan penghapusan.

```bash
ls -lh *.aud
```

#### 3. Cleanup Audit Files (.aud)
Menghapus file audit yang sudah lebih dari 7 hari untuk menjaga penggunaan storage tetap optimal.

```bash
find . -type f -name "*.aud" -mtime +7 -exec rm -f {} \;
```

---

### Trace & Log Cleanup

#### 4. Navigate to Trace Directory
Masuk ke direktori trace Oracle secara bertahap untuk memastikan tidak salah lokasi.

```bash
cd $ORACLE_BASE/diag/rdbms
ls
cd <db_name>
cd <instance_name>/trace
```

#### 5. Validate Files
Melihat file trace dan log yang tersedia sebelum dilakukan pembersihan.

```bash
ls -lh *.trc
ls -lh *.log
```

#### 6. Cleanup Trace Files (.trc)
Menghapus file trace lama yang sudah tidak digunakan untuk troubleshooting.

```bash
find . -type f -name "*.trc" -mtime +7 -exec rm -f {} \;
```

#### 7. Cleanup Log Files (.log)
Menghapus file log lama untuk menghindari penumpukan file yang tidak diperlukan.

```bash
find . -type f -name "*.log" -mtime +7 -exec rm -f {} \;
```

---

### Validation & Safety

#### 8. Preview Before Delete (Recommended)
Melakukan pengecekan terlebih dahulu file yang akan dihapus tanpa langsung mengeksekusi perintah delete.

```bash
find . -type f -mtime +7
```

#### 9. Disk Usage Check
Memastikan penggunaan storage setelah housekeeping dilakukan.

```bash
du -sh .
df -h
```

---

### Optional

#### 10. Compress Before Deletion
Melakukan kompresi file lama sebagai alternatif sebelum dilakukan penghapusan.

```bash
find . -type f -mtime +3 -exec gzip {} \;
```

---

## Summary

- Housekeeping dilakukan secara manual untuk memastikan kontrol penuh terhadap sistem  
- Setiap langkah diawali dengan validasi untuk menghindari kesalahan penghapusan  
- Retention digunakan sebagai acuan dalam menentukan file yang akan dibersihkan  
- Metode ini fleksibel dan dapat digunakan di berbagai environment Oracle  
- Aman digunakan untuk kebutuhan production system  
