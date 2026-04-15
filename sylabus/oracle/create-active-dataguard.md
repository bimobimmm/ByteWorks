# Oracle Active Data Guard + Broker + Auto Failover (ByteWorks)

## Environment

| Parameter | Value |
|----------|------|
| ORACLE_SID | BYTEWORK |
| ORACLE_HOME | /data/u01/app/oracle/product/19.0.0/dbhome_19 |
| DB_UNIQUE_NAME (Primary) | ByteWorks |
| DB_UNIQUE_NAME (Standby) | ByteWorks_standby |
| Archive Location | /data/archivelog/ByteWorks |

---

# PART 1 — PRIMARY SERVER

```sql
$ORACLE_HOME/bin/rman target / \
log=/data/scripts/oracle/backupdb.log << BKUP

run{
    delete noprompt backup;
    delete noprompt copy;

    allocate channel d1 device type disk;
    allocate channel d2 device type disk;
    allocate channel d3 device type disk;

    backup as compressed backupset
    database format='/data/backup/oracle/OCBCNISP/copy/%U';

    sql "alter system switch logfile";
    sql "alter system switch logfile";
    sql "alter system switch logfile";

    backup as compressed backupset
    archivelog all format='/data/backup/oracle/OCBCNISP/backupset/%U';
}

exit;
BKUP
```


## 1. Create Standby Redo Log

```sql
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 10 '/data/oradata/ByteWorks/redo10.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 11 '/data/oradata/ByteWorks/redo11.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 12 '/data/oradata/ByteWorks/redo12.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 13 '/data/oradata/ByteWorks/redo13.log' SIZE 200M;
```

---

## 2. Database Configuration

```sql
ALTER SYSTEM SET dg_broker_start=TRUE SCOPE=BOTH;
ALTER SYSTEM SET log_archive_max_processes=30 SCOPE=BOTH;

ALTER SYSTEM SET db_recovery_file_dest_size=5G;
ALTER SYSTEM SET db_recovery_file_dest='/data/archivelog/ByteWorks';

ALTER SYSTEM SET standby_file_management=AUTO SCOPE=BOTH;
ALTER SYSTEM SET os_authent_prefix='' SCOPE=SPFILE;
```

---

## 3. Restart Database

```sql
SHUTDOWN IMMEDIATE;
STARTUP;
```

---

## 4. Flashback & Logging

```sql
ALTER SYSTEM SET db_flashback_retention_target=30;
ALTER DATABASE FORCE LOGGING;
ALTER DATABASE FLASHBACK ON;
```

---

## 5. Create Standby Controlfile

```sql
ALTER DATABASE CREATE STANDBY CONTROLFILE AS '/data/backup/controlfile/bytework_stby.ctl';
```

---

## 6. Transfer Controlfile

```bash
scp /data/backup/controlfile/bytework_stby.ctl oracle@STANDBY:/data/backup/controlfile/
```

## STANDBY
di standby, masuk ke path oradata biasanya ada di /u01/oradata lalu cp standby controlfile td

```bash
cp /data/backup/controlfile/bytework_stby.ctl control01.ctl
cp control01.ctl control02.ctl
```

---

# PART 2 — NETWORK CONFIGURATION
## 7. listener.ora

```ini
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = PRIMARY_HOST)(PORT = 1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = BYTEWORK_DGMGRL)
      (ORACLE_HOME = /data/u01/app/oracle/product/19.0.0/dbhome_19)
      (SID_NAME = BYTEWORK)
    )
  )
```

---

## 8. tnsnames.ora

```ini
BYTEWORK_PRIMARY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = PRIMARY_HOST)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = BYTEWORK)
    )
  )

BYTEWORK_DGMGRL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = PRIMARY_HOST)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = BYTEWORK)
    )
  )

BYTEWORK_STANDBY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = STANDBY_HOST)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = BYTEWORK_STANDBY)
    )
  )
```

---

# PART 3 — STANDBY SERVER

---

## 9. Nyalakan database ke level mount 

```sql
STARTUP MOUNT 
```
## 10. Set DB Unique Name

```sql
ALTER SYSTEM SET db_unique_name='ByteWorks_standby' SCOPE=SPFILE;
```


```sql
SHUTDOWN IMMEDIATE;
```

```sql
STARTUP MOUNT;
```

---

## 11. Restore Database

```bash
$ORACLE_HOME/bin/rman target / <<EOF
run {
 allocate channel d1 device type disk;
 allocate channel d2 device type disk;
 allocate channel d3 device type disk;

 restore database from service BYTEWORK_PRIMARY;
 recover database from service BYTEWORK_PRIMARY;
}
exit;
EOF
```

---

## 12. Standby Redo Log (DROP & RECREATE)

```sql
SELECT GROUP#, STATUS FROM V$STANDBY_LOG;

ALTER SYSTEM SET standby_file_management=MANUAL SCOPE=BOTH;

ALTER DATABASE DROP STANDBY LOGFILE GROUP 10;
ALTER DATABASE DROP STANDBY LOGFILE GROUP 11;
ALTER DATABASE DROP STANDBY LOGFILE GROUP 12;
ALTER DATABASE DROP STANDBY LOGFILE GROUP 13;

ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 10 '/data/oradata/ByteWorks_standby/redo10.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 11 '/data/oradata/ByteWorks_standby/redo11.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 12 '/data/oradata/ByteWorks_standby/redo12.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 13 '/data/oradata/ByteWorks_standby/redo13.log' SIZE 200M;

ALTER SYSTEM SET standby_file_management=AUTO SCOPE=BOTH;
```

---

## 13. Open Standby

```sql
ALTER DATABASE OPEN READ ONLY;
```

---

# PART 4 — BROKER

## 14. Create Configuration

```bash
dgmgrl sys@BYTEWORK_PRIMARY
```

```sql
CREATE CONFIGURATION ByteWorks_DG
AS PRIMARY DATABASE IS ByteWorks
CONNECT IDENTIFIER IS BYTEWORK_PRIMARY;

ADD DATABASE ByteWorks_standby
AS CONNECT IDENTIFIER IS BYTEWORK_STANDBY
MAINTAINED AS PHYSICAL;

ENABLE CONFIGURATION;
ENABLE DATABASE ByteWorks_standby;
```

---

## 15. Enable Apply

```sql
EDIT DATABASE ByteWorks_standby SET STATE=APPLY-ON;
```

---

# PART 5 — AUTO FAILOVER

## 16. Configure FSFO

```sql
EDIT DATABASE ByteWorks SET PROPERTY LogXptMode='SYNC';
EDIT DATABASE ByteWorks_standby SET PROPERTY LogXptMode='SYNC';

EDIT DATABASE ByteWorks SET PROPERTY Binding='MANDATORY';
EDIT DATABASE ByteWorks_standby SET PROPERTY Binding='MANDATORY';

EDIT DATABASE ByteWorks SET PROPERTY FastStartFailoverTarget='ByteWorks_standby';
EDIT DATABASE ByteWorks_standby SET PROPERTY FastStartFailoverTarget='ByteWorks';

EDIT CONFIGURATION SET PROTECTION MODE AS MAXAVAILABILITY;

EDIT CONFIGURATION SET PROPERTY FastStartFailoverThreshold=30;
EDIT CONFIGURATION SET PROPERTY FastStartFailoverLagLimit=30;
EDIT CONFIGURATION SET PROPERTY FastStartFailoverAutoReinstate=TRUE;
EDIT CONFIGURATION SET PROPERTY ObserverReconnect=15;

ENABLE FAST_START FAILOVER;
```

---

# PART 6 — OBSERVER (SERVER KE-3)

## 17. Start Observer

```bash
dgmgrl sys@BYTEWORK_PRIMARY
```

```sql
START OBSERVER BYTEWORK_OBS 
IN BACKGROUND 
FILE IS '/data/observer/bytework/bytework.dat' 
LOGFILE IS '/data/observer/bytework/bytework.log' 
CONNECT IDENTIFIER IS BYTEWORK_PRIMARY;
```

---

# PART 7 — VALIDATION

```sql
SHOW CONFIGURATION;
SHOW DATABASE ByteWorks;
SHOW DATABASE ByteWorks_standby;
SHOW FAST_START FAILOVER;
```

---

# SUMMARY

- Standby database dibangun menggunakan RMAN restore from service  
- Standby controlfile dibuat dari primary dan di-restore di standby  
- Standby redo log dibuat di primary dan direcreate di standby untuk menghindari conflict  
- Data Guard Broker digunakan untuk manajemen konfigurasi  
- Replikasi menggunakan mode SYNC dengan protection MAXAVAILABILITY  
- Fast-Start Failover (FSFO) diaktifkan untuk failover otomatis  
- Observer dijalankan di server ke-3 untuk memastikan high availability  
- Sistem siap digunakan untuk kebutuhan production environment  
