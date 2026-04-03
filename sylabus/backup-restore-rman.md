<h1>How To Backup Restore On Oracle 19C RMAN @ByteWorks</h1>

<p>Step 1 - Open DBCA dan Pilih Create Database</p>
<img src="../images/instalation-db/1.png" width="600"/>

# 🛠️ Oracle RMAN Backup Script (ByteWorks)

## 📌 Overview
Script ini digunakan untuk melakukan **full backup (incremental level 0)** database Oracle menggunakan RMAN.  
Backup disimpan otomatis berdasarkan tanggal, termasuk:
- Database
- Archivelog
- Controlfile (current & standby)

---

## ⚙️ Script Backup

```bash
#!/bin/bash
# =============================================================
# Oracle RMAN Backup Script with Auto-Dated Directory
# Author  : bimoanggorojatii@gmail.com
# Project : ByteWorks
# =============================================================

# --- Konfigurasi dasar environment ---
export ORACLE_SID=BYTEWORK
export ORACLE_BASE=/data/u01/app/oracle
export ORACLE_HOME=/data/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_HOSTNAME=PLVBIFDBD101
export PATH=$PATH:$ORACLE_HOME/bin

# --- Format tanggal ---
DATE_DIR=$(date +%Y-%m-%d)
DATETIME=$(date +%d%m%y_%H%M%S)

# --- Direktori ---
BACKUP_BASE="/home/oracle/backup/BYTEWORK"
BACKUP_DIR="${BACKUP_BASE}/${DATE_DIR}"
LOG_DIR="/home/oracle/backup/log/BYTEWORK"

# --- Create directory ---
mkdir -p "${BACKUP_DIR}"
mkdir -p "${LOG_DIR}"

echo "Backup started at $(date)"

# --- RMAN Backup ---
${ORACLE_HOME}/bin/rman target=/ log="${LOG_DIR}/backup_${DATETIME}.log" <<EOF
run
{
  crosscheck backup;
  crosscheck archivelog all;
  delete noprompt expired archivelog all;
  delete noprompt expired backup;

  backup as compressed backupset incremental level 0 database
    format '${BACKUP_DIR}/data_L0_%d_%s_%p_%c_%T.bkp';

  backup as compressed backupset incremental level 0 archivelog all
    format '${BACKUP_DIR}/arch_L0_%d_%s_%p_%c_%T.bak';

  backup current controlfile for standby
    format '${BACKUP_DIR}/standby_ctl_L0_%d_%s_%p_%c_%T.bkp';

  backup current controlfile
    format '${BACKUP_DIR}/current_ctl_L0_%d_%s_%p_%c_%T.bkp';
}
EXIT;
EOF

# --- Status ---
if [ $? -eq 0 ]; then
  echo "Backup SUCCESS at $(date)"
else
  echo "Backup FAILED at $(date)"
fi
```

---

## ▶️ Cara Menjalankan

```bash
chmod +x backup.sh
./backup.sh
```

---

## 📂 Output Backup

- 📁 Backup Directory:
```
/home/oracle/backup/BYTEWORK/YYYY-MM-DD/
```

- 📄 Log File:
```
/home/oracle/backup/log/BYTEWORK/backup_TIMESTAMP.log
```

---

## ⚠️ Notes

- Menggunakan **incremental level 0 (full backup)**
- Sudah termasuk:
  - Database
  - Archivelog
  - Controlfile
- Script otomatis:
  - Crosscheck backup
  - Hapus expired backup

---
