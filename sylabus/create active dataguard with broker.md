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

## 6. Network Configuration

### listener.ora (Primary)

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

### tnsnames.ora

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

## 7. Prepare Standby

```sql
ALTER SYSTEM SET db_unique_name='ByteWorks_standby' SCOPE=SPFILE;
```

---

## 8. Create Standby Controlfile

```sql
ALTER DATABASE CREATE STANDBY CONTROLFILE AS '/data/backup/controlfile/bytework_stby.ctl';
```

Copy ke standby:

```bash
scp /data/backup/controlfile/bytework_stby.ctl oracle@STANDBY:/data/oradata/ByteWorks_standby/
```

---

## 9. Restore Database (Standby)

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

## 10. Recreate Standby Redo Log (Standby)

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

## 11. Open Standby

```sql
ALTER DATABASE OPEN READ ONLY;
```

---

## 12. Data Guard Broker Configuration

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

## 13. Configure Apply

```sql
EDIT DATABASE ByteWorks_standby SET STATE=APPLY-ON;
```

---

## 14. Auto Failover Configuration

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

## 15. Start Observer

```sql
START OBSERVER;
```

---

## 16. Validation

```sql
SHOW CONFIGURATION;
SHOW DATABASE ByteWorks;
SHOW DATABASE ByteWorks_standby;
SHOW FAST_START FAILOVER;
```

---

## Summary

- Mode: Active Data Guard + Broker  
- Failover: Automatic (FSFO enabled)  
- Protection Mode: Maximum Availability  
- Observer: Required for auto failover  
