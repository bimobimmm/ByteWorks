# Oracle User Management

Modul ini berisi langkah-langkah dasar mengelola user di Oracle Database, mulai dari membuat user biasa hingga memberikan hak akses DBA.

---

# 1. Login sebagai SYSDBA

```sql
sqlplus / as sysdba
```

atau

```sql
sqlplus sys/password@ORCL as sysdba
```

Cek user yang sedang digunakan.

```sql
SHOW USER;
```

---

# 2. Melihat Daftar User

```sql
SELECT username,
       account_status,
       default_tablespace
FROM dba_users
ORDER BY username;
```

---

# 3. Membuat User Baru

```sql
CREATE USER trainee
IDENTIFIED BY Oracle123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 100M ON users;
```

## Penjelasan

| Parameter | Fungsi |
|-----------|---------|
| IDENTIFIED BY | Password user |
| DEFAULT TABLESPACE | Tablespace default |
| TEMPORARY TABLESPACE | Tablespace temporary |
| QUOTA | Batas penggunaan tablespace |

---

# 4. Memberikan Hak Login

```sql
GRANT CREATE SESSION TO trainee;
```

> Tanpa privilege **CREATE SESSION**, user tidak dapat login.

---

# 5. Login Menggunakan User Baru

```sql
sqlplus trainee/Oracle123
```

atau

```sql
CONNECT trainee/Oracle123
```

---

# 6. Memberikan Hak Membuat Table

Login kembali sebagai SYS.

```sql
GRANT CREATE TABLE TO trainee;
```

---

# 7. Membuat Table

Login sebagai `trainee`.

```sql
CREATE TABLE employee (

    id      NUMBER PRIMARY KEY,
    nama    VARCHAR2(100),
    divisi  VARCHAR2(50),
    gaji    NUMBER

);
```

Insert data.

```sql
INSERT INTO employee
VALUES (1,'Bimo','Database',10000000);

COMMIT;
```

Cek data.

```sql
SELECT *
FROM employee;
```

---

# 8. Membuat User Read Only

```sql
CREATE USER report_user
IDENTIFIED BY Oracle123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 0M ON users;
```

Berikan hak login.

```sql
GRANT CREATE SESSION TO report_user;
```

> QUOTA 0M berarti user tidak dapat membuat object.

---

# 9. Memberikan Hak SELECT

```sql
GRANT SELECT
ON trainee.employee
TO report_user;
```

---

# 10. Test User Read Only

Login.

```sql
sqlplus report_user/Oracle123
```

Jalankan query.

```sql
SELECT *
FROM trainee.employee;
```

✅ Berhasil.

---

# 11. Test INSERT

```sql
INSERT INTO trainee.employee
VALUES (2,'Andi','IT',5000000);
```

Hasil

```
ORA-01031: insufficient privileges
```

---

# 12. Test UPDATE

```sql
UPDATE trainee.employee
SET nama='TEST'
WHERE id=1;
```

Hasil

```
ORA-01031
```

---

# 13. Test DELETE

```sql
DELETE
FROM trainee.employee;
```

Hasil

```
ORA-01031
```

---

# 14. Test CREATE TABLE

```sql
CREATE TABLE test(
    id NUMBER
);
```

Hasil

```
ORA-01031
```

---

# 15. Melihat Privilege User

System Privilege

```sql
SELECT *
FROM dba_sys_privs
WHERE grantee='REPORT_USER';
```

Object Privilege

```sql
SELECT owner,
       table_name,
       privilege
FROM dba_tab_privs
WHERE grantee='REPORT_USER';
```

---

# 16. Memberikan SELECT ke Banyak Table

```sql
GRANT SELECT ON trainee.employee TO report_user;
GRANT SELECT ON trainee.department TO report_user;
GRANT SELECT ON trainee.salary TO report_user;
```

---

# 17. Grant SELECT ke Seluruh Table dalam Schema

```sql
BEGIN

    FOR t IN (

        SELECT table_name
        FROM dba_tables
        WHERE owner='TRAINEE'

    )

    LOOP

        EXECUTE IMMEDIATE
            'GRANT SELECT ON TRAINEE.' ||
            t.table_name ||
            ' TO REPORT_USER';

    END LOOP;

END;
/
```

---

# 18. Membuat User Aplikasi

```sql
CREATE USER appuser
IDENTIFIED BY Oracle123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 500M ON users;
```

```sql
GRANT CREATE SESSION,
      CREATE TABLE,
      CREATE VIEW,
      CREATE SEQUENCE,
      CREATE PROCEDURE
TO appuser;
```

---

# 19. Membuat User DBA

```sql
CREATE USER dbadmin
IDENTIFIED BY Oracle123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA UNLIMITED ON users;
```

Berikan role DBA.

```sql
GRANT DBA TO dbadmin;
```

Verifikasi.

```sql
SELECT *
FROM dba_role_privs
WHERE grantee='DBADMIN';
```

---

# 20. Mengubah Password

```sql
ALTER USER trainee
IDENTIFIED BY Oracle456;
```

---

# 21. Lock User

```sql
ALTER USER trainee ACCOUNT LOCK;
```

---

# 22. Unlock User

```sql
ALTER USER trainee ACCOUNT UNLOCK;
```

---

# 23. Menghapus User

```sql
DROP USER report_user CASCADE;
```

---

# Ringkasan Privileges

| Privilege | Fungsi |
|------------|--------|
| CREATE SESSION | Login ke database |
| CREATE TABLE | Membuat table |
| CREATE VIEW | Membuat view |
| CREATE PROCEDURE | Membuat procedure |
| CREATE SEQUENCE | Membuat sequence |
| CREATE TRIGGER | Membuat trigger |
| SELECT | Membaca data |
| INSERT | Menambah data |
| UPDATE | Mengubah data |
| DELETE | Menghapus data |
| EXECUTE | Menjalankan procedure/function |
| DBA | Administrator penuh |

---

# Best Practice

- Jangan memberikan role **DBA** ke user aplikasi.
- Gunakan **Principle of Least Privilege**.
- Berikan hanya privilege yang benar-benar dibutuhkan.
- Pisahkan user **Application**, **Reporting**, dan **Administrator**.
- Gunakan object privilege (`GRANT SELECT ON schema.table`) dibanding privilege yang terlalu luas.

---

**Author:** noxiousroots
