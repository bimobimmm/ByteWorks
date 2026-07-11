Oracle 19c Data Guard Observer (FSFO) Setup SOP

1. Verify Oracle Environment

    echo $ORACLE_HOME
    dgmgrl -version

------------------------------------------------------------------------

2. Configure tnsnames.ora

Edit:

    vi $ORACLE_HOME/network/admin/tnsnames.ora

Example:

    dbbni =
     (DESCRIPTION =
       (ADDRESS=(PROTOCOL=TCP)(HOST=exaprimary)(PORT=1521))
       (CONNECT_DATA=
         (SERVICE_NAME=DBBNI_DGMGRL)
       )
     )

    dbbni_standby =
     (DESCRIPTION =
       (ADDRESS=(PROTOCOL=TCP)(HOST=exastandby)(PORT=1521))
       (CONNECT_DATA=
         (SERVICE_NAME=DBBNI_STANDBY_DGMGRL)
       )
     )

Test:

    tnsping dbbni
    tnsping dbbni_standby

------------------------------------------------------------------------

3. Create Wallet

    mkdir -p /u01/observer/dbbni/wallet

    mkstore -wrl /u01/observer/dbbni/wallet -create

------------------------------------------------------------------------

4. Create sqlnet.ora

    vi $ORACLE_HOME/network/admin/sqlnet.ora

Contents:

    NAMES.DIRECTORY_PATH=(TNSNAMES,EZCONNECT)

    WALLET_LOCATION =
     (SOURCE =
       (METHOD = FILE)
       (METHOD_DATA =
         (DIRECTORY = /u01/observer/dbbni/wallet)
       )
     )

    SQLNET.WALLET_OVERRIDE=TRUE

------------------------------------------------------------------------

5. Add Wallet Credentials

Check DGConnectIdentifier:

    SHOW DATABASE VERBOSE dbbni;
    SHOW DATABASE VERBOSE dbbni_standby;

Create credentials matching DGConnectIdentifier:

    mkstore -wrl /u01/observer/dbbni/wallet -createCredential dbbni SYS
    mkstore -wrl /u01/observer/dbbni/wallet -createCredential dbbni_standby SYS

Verify:

    mkstore -wrl /u01/observer/dbbni/wallet -listCredential

------------------------------------------------------------------------

6. Test Wallet

    sqlplus /@dbbni as sysdba
    sqlplus /@dbbni_standby as sysdba

Both must connect without asking for a password.

------------------------------------------------------------------------

7. Start Observer

    dgmgrl sys/password@dbbni

    START OBSERVER DBBNI_OBS
    IN BACKGROUND
    FILE IS '/u01/observer/dbbni/dbbni.dat'
    LOGFILE IS '/u01/observer/dbbni/dbbni.log'
    CONNECT IDENTIFIER IS dbbni;

------------------------------------------------------------------------

8. Verify

    SHOW OBSERVER;
    SHOW FAST_START FAILOVER;
    SHOW CONFIGURATION;

------------------------------------------------------------------------

9. Monitor

    tail -f /u01/observer/dbbni/dbbni.log
    ps -ef | grep dgmgrl

------------------------------------------------------------------------

Troubleshooting

ORA-12578

Cause: - sqlnet.ora missing or incorrect - Wallet cannot be opened -
Wrong permissions - Auto-login wallet missing

Fix: - Verify sqlnet.ora - Verify cwallet.sso - Verify permissions -
Recreate auto-login wallet if necessary

ORA-01017

Cause: - Incorrect SYS password stored in wallet

Fix:

    mkstore -wrl /u01/observer/dbbni/wallet -modifyCredential dbbni SYS

DGM-16979 Authentication failed

Cause: - Wallet missing credential for DGConnectIdentifier - Password
file mismatch - Wrong TNS alias - DGConnectIdentifier mismatch

ORA-16820

Cause: - Observer stopped monitoring.

Fix: - Restart Observer.

------------------------------------------------------------------------

Pre-Start Checklist

-   ☐ tnsping dbbni OK
-   ☐ tnsping dbbni_standby OK
-   ☐ sqlplus /@dbbni as sysdba OK
-   ☐ sqlplus /@dbbni_standby as sysdba OK
-   ☐ Wallet contains all DGConnectIdentifier credentials
-   ☐ Password files synchronized
-   ☐ SHOW CONFIGURATION = SUCCESS
