===========================================================
                ORACLE DATABASE USER MANAGEMENT
              Beginner to Database Administrator (DBA)
===========================================================

Author  : Bimo Anggoro Jati
Category: Oracle Database Administration
Version : Oracle 19c / 21c / 23ai

===========================================================
1. LOGIN SEBAGAI SYSDBA
===========================================================

Login menggunakan user SYS.

sqlplus / as sysdba

atau

sqlplus sys/password@ORCL as sysdba

Cek user yang sedang digunakan

SQL> SHOW USER;

===========================================================
2. MELIHAT USER YANG ADA
===========================================================

SELECT username,
       account_status,
       default_tablespace
FROM dba_users
ORDER BY username;

===========================================================
3. MEMBUAT USER BARU
===========================================================

CREATE USER trainee
IDENTIFIED BY Oracle123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 100M ON users;

Penjelasan

IDENTIFIED BY      = Password user
DEFAULT TABLESPACE = Tablespace default
TEMPORARY TABLESPACE = Tablespace temporary
QUOTA              = Maksimum penggunaan tablespace

===========================================================
4. MEMBERIKAN HAK LOGIN
===========================================================

GRANT CREATE SESSION TO trainee;

Sekarang user sudah bisa login tetapi belum bisa melakukan apa pun.

===========================================================
5. CEK SYSTEM PRIVILEGES
===========================================================

SELECT *
FROM dba_sys_privs
WHERE grantee='TRAINEE';

===========================================================
6. LOGIN SEBAGAI USER BARU
===========================================================

sqlplus trainee/Oracle123

atau

CONNECT trainee/Oracle123

===========================================================
7. MEMBUAT TABLE OLEH USER TRAINEE
===========================================================

Kembali ke SYS

GRANT CREATE TABLE TO trainee;

Login sebagai trainee

CREATE TABLE employee(

    id NUMBER PRIMARY KEY,
    nama VARCHAR2(100),
    divisi VARCHAR2(50),
    gaji NUMBER

);

INSERT INTO employee VALUES(1,'Bimo','Database',10000000);

COMMIT;

Cek isi tabel

SELECT * FROM employee;

===========================================================
8. MEMBUAT USER REPORT (READ ONLY)
===========================================================

Kembali login SYS

CREATE USER report_user
IDENTIFIED BY Oracle123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 0M ON users;

GRANT CREATE SESSION TO report_user;

Catatan:

QUOTA 0M berarti user tidak boleh membuat object.

===========================================================
9. MEMBERIKAN HAK SELECT KE SATU TABLE
===========================================================

GRANT SELECT
ON trainee.employee
TO report_user;

===========================================================
10. TEST USER REPORT
===========================================================

Login

sqlplus report_user/Oracle123

Lakukan

SELECT * FROM trainee.employee;

Hasil

Berhasil.

===========================================================
11. TEST INSERT
===========================================================

INSERT INTO trainee.employee
VALUES(2,'Andi','IT',5000000);

Hasil

ORA-01031
Insufficient Privileges

Karena hanya diberikan SELECT.

===========================================================
12. TEST UPDATE
===========================================================

UPDATE trainee.employee
SET nama='TEST'
WHERE id=1;

Hasil

ORA-01031

===========================================================
13. TEST DELETE
===========================================================

DELETE
FROM trainee.employee;

Hasil

ORA-01031

===========================================================
14. TEST CREATE TABLE
===========================================================

CREATE TABLE test(

id NUMBER

);

Hasil

ORA-01031

===========================================================
15. CEK OBJECT PRIVILEGES
===========================================================

SELECT owner,
       table_name,
       privilege

FROM dba_tab_privs

WHERE grantee='REPORT_USER';

===========================================================
16. MEMBERIKAN HAK SELECT KE BANYAK TABLE
===========================================================

GRANT SELECT ON trainee.employee TO report_user;
GRANT SELECT ON trainee.department TO report_user;
GRANT SELECT ON trainee.salary TO report_user;

===========================================================
17. MEMBERIKAN SELECT KE SEMUA TABLE DALAM SCHEMA
===========================================================

BEGIN

FOR i IN (

SELECT table_name

FROM dba_tables

WHERE owner='TRAINEE'

)

LOOP

EXECUTE IMMEDIATE

'GRANT SELECT ON TRAINEE.'

|| i.table_name ||

' TO REPORT_USER';

END LOOP;

END;

/

===========================================================
18. MEMBERIKAN HAK CREATE VIEW
===========================================================

GRANT CREATE VIEW TO trainee;

===========================================================
19. MEMBUAT VIEW
===========================================================

CREATE VIEW employee_view AS

SELECT *

FROM employee;

===========================================================
20. MEMBERIKAN HAK SELECT KE VIEW
===========================================================

GRANT SELECT
ON trainee.employee_view
TO report_user;

===========================================================
21. MEMBUAT USER APPLICATION
===========================================================

CREATE USER appuser

IDENTIFIED BY Oracle123

DEFAULT TABLESPACE users

TEMPORARY TABLESPACE temp

QUOTA 500M ON users;

GRANT CREATE SESSION,
      CREATE TABLE,
      CREATE VIEW,
      CREATE SEQUENCE,
      CREATE PROCEDURE

TO appuser;

===========================================================
22. MEMBUAT USER DBA
===========================================================

CREATE USER dbadmin

IDENTIFIED BY Oracle123

DEFAULT TABLESPACE users

TEMPORARY TABLESPACE temp

QUOTA UNLIMITED ON users;

===========================================================
23. MEMBERIKAN ROLE DBA
===========================================================

GRANT DBA TO dbadmin;

===========================================================
24. VERIFIKASI ROLE
===========================================================

SELECT *

FROM dba_role_privs

WHERE grantee='DBADMIN';

===========================================================
25. VERIFIKASI SYSTEM PRIVILEGES
===========================================================

SELECT *

FROM dba_sys_privs

WHERE grantee='DBADMIN';

===========================================================
26. MELIHAT SEMUA ROLE USER
===========================================================

SELECT *

FROM dba_role_privs

WHERE grantee='TRAINEE';

===========================================================
27. MELIHAT SEMUA OBJECT PRIVILEGES
===========================================================

SELECT *

FROM dba_tab_privs

WHERE grantee='REPORT_USER';

===========================================================
28. MENGUBAH PASSWORD USER
===========================================================

ALTER USER trainee
IDENTIFIED BY Oracle456;

===========================================================
29. LOCK USER
===========================================================

ALTER USER trainee ACCOUNT LOCK;

===========================================================
30. UNLOCK USER
===========================================================

ALTER USER trainee ACCOUNT UNLOCK;

===========================================================
31. DROP USER
===========================================================

DROP USER report_user CASCADE;

===========================================================
32. DROP USER BESERTA OBJECT
===========================================================

DROP USER trainee CASCADE;

===========================================================
RINGKASAN PRIVILEGES
===========================================================

CREATE SESSION
    Login ke database

CREATE TABLE
    Membuat tabel

CREATE VIEW
    Membuat view

CREATE SEQUENCE
    Membuat sequence

CREATE PROCEDURE
    Membuat procedure

CREATE TRIGGER
    Membuat trigger

CREATE SYNONYM
    Membuat synonym

CREATE TYPE
    Membuat object type

UNLIMITED TABLESPACE
    Tidak dibatasi quota tablespace

SELECT
    Membaca data

INSERT
    Menambah data

UPDATE
    Mengubah data

DELETE
    Menghapus data

EXECUTE
    Menjalankan Procedure/Function

DBA
    Hak administrator penuh

===========================================================
BEST PRACTICE
===========================================================

✓ Jangan memberikan role DBA ke user aplikasi.
✓ Gunakan principle of least privilege.
✓ Berikan CREATE SESSION terlebih dahulu.
✓ Gunakan object privilege (GRANT SELECT ON schema.table).
✓ Hindari GRANT ALL PRIVILEGES kecuali benar-benar diperlukan.
✓ Gunakan role khusus untuk mempermudah manajemen privilege.
✓ Pisahkan user aplikasi, user reporting, dan user administrator.

===========================================================
END OF DOCUMENT
===========================================================
