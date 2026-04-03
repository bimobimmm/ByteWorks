# Oracle RMAN Backup & Restore Guide

## 1. Environment

| Parameter        | Value |
|-----------------|------|
| ORACLE_SID      | BYTEWORK |
| ORACLE_HOME     | /data/u01/app/oracle/product/19.0.0/dbhome_19 |
| Backup Location | /home/oracle/backup/BYTEWORK |
| Log Location    | /home/oracle/backup/log/BYTEWORK |

---

## 2. Backup Database

### 2.1 Backup Script

```bash
#!/bin/bash
# =============================================================
# Oracle RMAN Backup Script with Auto-Dated Directory
# Author  : bimoanggorojatii@gmail.com
# Project : ByteWorks
# =============================================================

export ORACLE_SID=BYTEWORK
export ORACLE_BASE=/data/u01/app/oracle
export ORACLE_HOME=/data/u01/app/oracle/product/19.0.0/dbhome_19
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

### 2.2 Execute Backup

```bash
chmod +x backup.sh
./backup.sh
```

---

### 2.3 Enable Archivelog (If Required)

```sql
startup mount;
alter database archivelog;
archive log list;
alter database open;
```

---

### 2.4 Generate PFILE

```sql
CREATE PFILE FROM SPFILE;
```

Location:
```
$ORACLE_HOME/dbs
```

---

## 3. Prepare Restore (Primary to Standby)

### 3.1 Copy Backup

```bash
scp -r /home/oracle/backup/BYTEWORK/YYYY-MM-DD oracle@STANDBY_IP:/home/oracle/backup/BYTEWORK
```

---

### 3.2 Copy PFILE

```bash
scp $ORACLE_HOME/dbs/initBYTEWORK.ora oracle@STANDBY_IP:$ORACLE_HOME/dbs
```

---

## 4. Prepare Standby

### 4.1 Create Required Directories

```bash
mkdir -p /data/u01/app/oracle/admin/BYTEWORK/adump
mkdir -p /data/u01/app/oracle/recovery_area
```

---

### 4.2 Startup Nomount

```sql
startup nomount pfile='$ORACLE_HOME/dbs/initBYTEWORK.ora';
```

---

## 5. Restore Controlfile

```rman
restore controlfile from '/home/oracle/backup/BYTEWORK/YYYY-MM-DD/current_ctl_L0_*.bkp';
alter database mount;
```

---

## 6. Register Backup

```rman
crosscheck backup;
catalog start with '/home/oracle/backup/BYTEWORK/YYYY-MM-DD';
```

---

## 7. Restore Database

### 7.1 Restore Script

```bash
#!/bin/bash

export ORACLE_SID=BYTEWORK
export ORACLE_BASE=/data/u01/app/oracle
export ORACLE_HOME=/data/u01/app/oracle/product/19.0.0/dbhome_19
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

### 7.2 Execute Restore

```bash
chmod +x restore.sh
./restore.sh
```

---

## 8. Post-Restore

```sql
create spfile from pfile;
create pfile from spfile;
```

---

## 9. Summary

- Backup type: RMAN Incremental Level 0 (Full)
- Components: Database, Archivelog, Controlfile
- Restore method: Full restore with recovery and resetlogs
- Environment: BYTEWORK
