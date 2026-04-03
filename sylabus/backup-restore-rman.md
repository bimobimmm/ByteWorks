# 🛠️ Oracle RMAN Backup & Restore Guide (ByteWorks)

## 📌 Environment

- ORACLE_SID      : BYTEWORK  
- ORACLE_HOME     : /data/u01/app/oracle/product/19.0.0/dbhome_1  
- Backup Location : /home/oracle/backup/BYTEWORK  
- Log Location    : /home/oracle/backup/log/BYTEWORK  

---

# 🔹 PART 1 — BACKUP DATABASE

## ⚙️ Script Backup

```bash
#!/bin/bash
# =============================================================
# Oracle RMAN Backup Script with Auto-Dated Directory
# Author  : bimoanggorojatii@gmail.com
# Project : ByteWorks
# =============================================================

export ORACLE_SID=BYTEWORK
export ORACLE_BASE=/data/u01/app/oracle
export ORACLE_HOME=/data/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_HOSTNAME=PLVBIFDBD101
export PATH=$PATH:$ORACLE_HOME/bin

DATE_DIR=$(date +%Y-%m-%d)
DATETIME=$(date +%d%m%y_%H%M%S)

BACKUP_BASE="/home/oracle/backup/BYTEWORK"
BACKUP_DIR="${BACKUP_BASE}/${DATE_DIR}"
LOG_DIR="/home/oracle/backup/log/BYTEWORK"

mkdir -p "${BACKUP_DIR}"
mkdir -p "${LOG_DIR}"

echo "Backup started at $(date)"

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

if [ $? -eq 0 ]; then
  echo "Backup SUCCESS"
else
  echo "Backup FAILED"
fi
```

---

## ▶️ Menjalankan Backup

```bash
chmod +x backup.sh
./backup.sh
```

---

## ⚠️ Jika Database Belum ARCHIVELOG

```sql
startup mount;
alter database archivelog;
archive log list;
alter database open;
```

---

## 📄 Generate PFILE

```sql
CREATE PFILE FROM SPFILE;
```

Lokasi:
```
$ORACLE_HOME/dbs
```

---

# 🔹 PART 2 — PREPARE RESTORE (PRIMARY → STANDBY)

## 📦 Copy Backup

```bash
scp -r /home/oracle/backup/BYTEWORK/YYYY-MM-DD oracle@STANDBY_IP:/home/oracle/backup/BYTEWORK
```

## 📄 Copy PFILE

```bash
scp $ORACLE_HOME/dbs/initBYTEWORK.ora oracle@STANDBY_IP:$ORACLE_HOME/dbs
```

---

# 🔹 PART 3 — PREPARE STANDBY

## 📂 Create Required Directory

```bash
mkdir -p /data/u01/app/oracle/admin/BYTEWORK/adump
mkdir -p /data/u01/app/oracle/recovery_area
```

---

## ⚙️ Startup Nomount

```sql
startup nomount pfile='$ORACLE_HOME/dbs/initBYTEWORK.ora';
```

---

## 🔹 PART 4 — RESTORE CONTROLFILE

```rman
restore controlfile from '/home/oracle/backup/BYTEWORK/YYYY-MM-DD/current_ctl_L0_*.bkp';
alter database mount;
```

---

## 🔹 PART 5 — REGISTER BACKUP

```rman
crosscheck backup;
catalog start with '/home/oracle/backup/BYTEWORK/YYYY-MM-DD';
```

---

# 🔹 PART 6 — SCRIPT RESTORE

## ⚙️ Restore Script

```bash
#!/bin/bash

export ORACLE_SID=BYTEWORK
export ORACLE_BASE=/data/u01/app/oracle
export ORACLE_HOME=/data/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_HOSTNAME=STANDBY_HOST
export PATH=$PATH:$ORACLE_HOME/bin

DATETIME=$(date +%d%m%y_%H%M%S)
LOGFILE="/home/oracle/backup/log/BYTEWORK/restore_${DATETIME}.log"

echo "Restore started at $(date)"

${ORACLE_HOME}/bin/rman target=/ log="${LOGFILE}" <<EOF
run {
  restore database;
  switch datafile all;
  switch tempfile all;
  recover database;
}
alter database open resetlogs;
exit;
EOF

if [ $? -eq 0 ]; then
  echo "Restore SUCCESS"
else
  echo "Restore FAILED"
fi
```

---

## ▶️ Jalankan Restore

```bash
chmod +x restore.sh
./restore.sh
```

---

# 🔹 PART 7 — POST RESTORE

## 🔧 Create SPFILE

```sql
create spfile from pfile;
create pfile from spfile;
```

# ✅ SUMMARY

✔ Backup: RMAN Level 0 (Full)  
✔ Include: DB + Archivelog + Controlfile  
✔ Restore: Full restore + recover + resetlogs  
✔ Environment: Consistent (BYTEWORK)

