# Oracle Active Data Guard + Broker + Auto Failover (ByteWorks)

## 1. Environment

| Parameter | Value |
|----------|------|
| ORACLE_SID | BYTEWORK |
| ORACLE_HOME | /data/u01/app/oracle/product/19.0.0/dbhome_19 |
| DB_UNIQUE_NAME (Primary) | ByteWorks |
| DB_UNIQUE_NAME (Standby) | ByteWorks_standby |
| Archive Location | /data/archivelog/ByteWorks |

---

## 2. Create Standby Redo Log (Primary)

```sql
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 10 '/data/oradata/ByteWorks/redo10.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 11 '/data/oradata/ByteWorks/redo11.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 12 '/data/oradata/ByteWorks/redo12.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 13 '/data/oradata/ByteWorks/redo13.log' SIZE 200M;
```

---

## 3. Primary Configuration

```sql
ALTER SYSTEM SET dg_broker_start=TRUE SCOPE=BOTH;
ALTER SYSTEM SET log_archive_max_processes=30 SCOPE=BOTH;

ALTER SYSTEM SET db_recovery_file_dest_size=5G;
ALTER SYSTEM SET db_recovery_file_dest='/data/archivelog/ByteWorks';

ALTER SYSTEM SET standby_file_management=AUTO SCOPE=BOTH;
ALTER SYSTEM SET os_authent_prefix='' SCOPE=SPFILE;
```

---

## 4. Restart Database

```sql
SHUTDOWN IMMEDIATE;
STARTUP;
```

---

## 5. Flashback Configuration

```sql
ALTER SYSTEM SET db_flashback_retention_target=30;
ALTER DATABASE FORCE LOGGING;
SELECT FLASHBACK_ON FROM V$DATABASE;
ALTER DATABASE FLASHBACK ON;
```

---

## 6. Listener Configuration

### Primary

```ini
LISTENER_BYTEWORK =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = PRIMARY_HOST)(PORT = 1521))
    )
  )

SID_LIST_LISTENER_BYTEWORK =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = BYTEWORK_DGMGRL)
      (ORACLE_HOME = /data/u01/app/oracle/product/19.0.0/dbhome_19)
      (SID_NAME = BYTEWORK)
    )
  )
```

---

### Standby

```ini
LISTENER_BYTEWORK =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = STANDBY_HOST)(PORT = 1521))
    )
  )

SID_LIST_LISTENER_BYTEWORK =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = BYTEWORK_STANDBY_DGMGRL)
      (ORACLE_HOME = /data/u01/app/oracle/product/19.0.0/dbhome_19)
      (SID_NAME = BYTEWORK)
    )
  )
```

---

## 7. TNS Configuration (Both Server)

```ini
BYTEWORK_PRIMARY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = PRIMARY_HOST)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = BYTEWORK)
    )
  )

BYTEWORK_DGMGRL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = PRIMARY_HOST)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = BYTEWORK)
    )
  )

BYTEWORK_STANDBY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = STANDBY_HOST)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = BYTEWORK_STANDBY)
    )
  )
```

---

## 8. Prepare Standby

```sql
ALTER SYSTEM SET db_unique_name='ByteWorks_standby' SCOPE=SPFILE;
```

---

## 9. Create & Transfer Standby Controlfile

```sql
ALTER DATABASE CREATE STANDBY CONTROLFILE AS '/data/backup/controlfile/bytework_stby.ctl';
```

```bash
scp /data/backup/controlfile/bytework_stby.ctl oracle@STANDBY:/data/oradata/ByteWorks_standby/
```

---

## 10. Restore Standby Controlfile

```sql
SHUTDOWN IMMEDIATE;
STARTUP NOMOUNT;
```

```rman
RESTORE STANDBY CONTROLFILE FROM '/data/oradata/ByteWorks_standby/bytework_stby.ctl';
```

```sql
ALTER DATABASE MOUNT;
```

---

## 11. Restore Database (Active Duplicate)

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

## 12. Recreate Standby Redo Log (Standby)

```sql
ALTER SYSTEM SET standby_file_management=MANUAL SCOPE=BOTH;

ALTER DATABASE DROP STANDBY LOGFILE GROUP 10;
ALTER DATABASE DROP STANDBY LOGFILE GROUP 11;
ALTER DATABASE DROP STANDBY LOGFILE GROUP 12;

ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 10 '/data/oradata/ByteWorks_standby/redo10.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 11 '/data/oradata/ByteWorks_standby/redo11.log' SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 12 '/data/oradata/ByteWorks_standby/redo12.log' SIZE 200M;

ALTER SYSTEM SET standby_file_management=AUTO SCOPE=BOTH;
```

---

## 13. Open Standby

```sql
ALTER DATABASE OPEN READ ONLY;
```

---

## 14. Data Guard Broker Configuration

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

## 16. Auto Failover Configuration

```sql
EDIT DATABASE ByteWorks SET PROPERTY LogXptMode='SYNC';
EDIT DATABASE ByteWorks SET PROPERTY Binding='MANDATORY';

EDIT DATABASE ByteWorks_standby SET PROPERTY LogXptMode='SYNC';
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

## 17. Start Observer (Server ke-3)

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

## 18. Validation

```sql
SHOW CONFIGURATION;
SHOW DATABASE ByteWorks;
SHOW DATABASE ByteWorks_standby;
SHOW FAST_START FAILOVER;
```

---

## Summary

- Standby controlfile wajib di-restore sebelum database restore  
- Standby redo log wajib ada di primary & standby  
- Broker mempermudah management ADG  
- Observer wajib untuk auto failover  
- Setup ini production-ready (enterprise level)  
