#### ORA-16475: succeeded with warnings

- Penyebab: standby redo log (SRL) tidak ada atau tidak sesuai
- Status: tidak blocking, tetapi **tidak direkomendasikan untuk diabaikan**

---

### Recommended Solution (Recreate Standby Redo Log)

Pendekatan terbaik adalah **drop dan recreate standby redo log**, dengan ukuran yang sama seperti redo log primary.

---

### Step 1 — Cek Redo Log Size (Primary)

```sql
SELECT GROUP#, BYTES/1024/1024 AS SIZE_MB FROM V$LOG;
```

Catat ukuran (misal: 200 MB)

---

### Step 2 — Cek Standby Redo Log (Standby)

```sql
SELECT GROUP#, BYTES/1024/1024 AS SIZE_MB FROM V$STANDBY_LOG;
```

---

### Step 3 — Drop Standby Redo Log Lama

```sql
ALTER DATABASE DROP STANDBY LOGFILE GROUP <GROUP#>;
```

Lakukan untuk semua standby redo log yang tidak sesuai.

---

### Step 4 — Recreate Standby Redo Log

Jumlah SRL minimal:
```
(Online Redo Log Group + 1)
```

Contoh:

```sql
ALTER DATABASE ADD STANDBY LOGFILE SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE SIZE 200M;
```

---

### Step 5 — Verifikasi

```sql
SELECT GROUP#, STATUS, BYTES/1024/1024 AS SIZE_MB FROM V$STANDBY_LOG;
```

Pastikan:
- Size sama dengan primary
- Status valid

---

### Notes

- Size standby redo log **harus sama atau lebih besar** dari primary
- Jumlah SRL ≥ (redo log group primary + 1)
- Best practice: gunakan ukuran yang sama untuk menghindari bottleneck
