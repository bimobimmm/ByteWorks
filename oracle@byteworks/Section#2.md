# Table Management

Table Management merupakan salah satu dasar dalam administrasi Oracle Database. Materi ini membahas cara membuat, mengubah, menghapus, dan mengelola tabel beserta struktur kolomnya.

---

# Daftar Isi

- Create Table
- Describe Table
- Insert Data
- Select Data
- Update Data
- Delete Data
- Truncate Table
- Alter Table
- Rename Table
- Drop Table
- Flashback Drop (Recycle Bin)
- Best Practice

---

# 1. Create Table

Membuat tabel baru.

```sql
CREATE TABLE employee (

    employee_id    NUMBER,
    employee_name  VARCHAR2(100),
    department     VARCHAR2(50),
    salary         NUMBER(10,2),
    created_date   DATE

);
```

---

# 2. Melihat Struktur Table

```sql
DESC employee;
```

atau

```sql
DESCRIBE employee;
```

---

# 3. Melihat Daftar Table

```sql
SELECT table_name
FROM user_tables;
```

Semua schema

```sql
SELECT owner,
       table_name
FROM dba_tables;
```

---

# 4. Insert Data

Menambahkan data ke dalam tabel.

```sql
INSERT INTO employee
VALUES
(
    1,
    'Bimo',
    'Database',
    10000000,
    SYSDATE
);

COMMIT;
```

Insert beberapa data.

```sql
INSERT INTO employee
VALUES
(
    2,
    'Andi',
    'Infrastructure',
    8500000,
    SYSDATE
);

INSERT INTO employee
VALUES
(
    3,
    'Sarah',
    'Application',
    9500000,
    SYSDATE
);

COMMIT;
```

---

# 5. Select Data

Menampilkan seluruh data.

```sql
SELECT *
FROM employee;
```

Menampilkan kolom tertentu.

```sql
SELECT employee_name,
       salary
FROM employee;
```

Menggunakan kondisi.

```sql
SELECT *
FROM employee
WHERE salary > 9000000;
```

---

# 6. Update Data

Mengubah data yang sudah ada.

```sql
UPDATE employee
SET salary = 12000000
WHERE employee_id = 1;

COMMIT;
```

Update beberapa kolom.

```sql
UPDATE employee
SET department = 'IT',
    salary = 13000000
WHERE employee_id = 2;

COMMIT;
```

---

# 7. Delete Data

Menghapus data tertentu.

```sql
DELETE
FROM employee
WHERE employee_id = 3;

COMMIT;
```

Menghapus seluruh data.

```sql
DELETE
FROM employee;

COMMIT;
```

---

# 8. Truncate Table

Menghapus seluruh data dengan cepat.

```sql
TRUNCATE TABLE employee;
```

### Perbedaan DELETE dan TRUNCATE

| DELETE | TRUNCATE |
|---------|-----------|
| Menghapus data per baris | Menghapus seluruh data sekaligus |
| Bisa menggunakan WHERE | Tidak bisa menggunakan WHERE |
| Bisa di-ROLLBACK sebelum COMMIT | Tidak bisa di-ROLLBACK |
| Lebih lambat | Lebih cepat |

---

# 9. Alter Table

## Menambah Kolom

```sql
ALTER TABLE employee
ADD email VARCHAR2(100);
```

---

## Mengubah Ukuran Kolom

```sql
ALTER TABLE employee
MODIFY employee_name VARCHAR2(200);
```

---

## Mengubah Tipe Data

```sql
ALTER TABLE employee
MODIFY salary NUMBER(15,2);
```

---

## Rename Kolom

```sql
ALTER TABLE employee
RENAME COLUMN employee_name
TO full_name;
```

---

## Drop Kolom

```sql
ALTER TABLE employee
DROP COLUMN email;
```

---

# 10. Rename Table

```sql
RENAME employee
TO employee_master;
```

---

# 11. Drop Table

Menghapus tabel beserta datanya.

```sql
DROP TABLE employee_master;
```

Jika terdapat Constraint.

```sql
DROP TABLE employee_master CASCADE CONSTRAINTS;
```

---

# 12. Flashback Drop

Melihat tabel yang masuk Recycle Bin.

```sql
SHOW RECYCLEBIN;
```

atau

```sql
SELECT *
FROM recyclebin;
```

Restore tabel.

```sql
FLASHBACK TABLE employee_master
TO BEFORE DROP;
```

Menghapus permanen.

```sql
PURGE TABLE employee_master;
```

Menghapus seluruh Recycle Bin.

```sql
PURGE RECYCLEBIN;
```

---

# 13. Melihat Struktur Tabel dari Data Dictionary

```sql
SELECT column_name,
       data_type,
       data_length,
       nullable
FROM user_tab_columns
WHERE table_name='EMPLOYEE';
```

---

# 14. Melihat Jumlah Record

```sql
SELECT COUNT(*)
FROM employee;
```

---

# 15. Best Practice

- Gunakan nama tabel yang konsisten (misalnya `employee`, `department`, `orders`).
- Tentukan tipe data yang sesuai untuk setiap kolom.
- Hindari penggunaan `VARCHAR2(4000)` jika tidak diperlukan.
- Selalu gunakan `COMMIT` setelah transaksi yang sudah dipastikan benar.
- Gunakan `TRUNCATE` untuk menghapus seluruh data jika tidak memerlukan rollback.
- Gunakan `DROP TABLE` hanya jika tabel memang sudah tidak dibutuhkan.
- Manfaatkan `FLASHBACK TABLE` untuk memulihkan tabel yang tidak sengaja dihapus (jika Recycle Bin masih aktif).

---

# Ringkasan

| Perintah | Fungsi |
|----------|--------|
| CREATE TABLE | Membuat tabel |
| DESC | Melihat struktur tabel |
| INSERT | Menambahkan data |
| SELECT | Menampilkan data |
| UPDATE | Mengubah data |
| DELETE | Menghapus data |
| TRUNCATE | Menghapus seluruh data |
| ALTER TABLE | Mengubah struktur tabel |
| RENAME | Mengganti nama tabel |
| DROP TABLE | Menghapus tabel |
| FLASHBACK TABLE | Mengembalikan tabel yang terhapus |
| PURGE | Menghapus permanen dari Recycle Bin |

---

**Author:** Bimo Anggoro Jati
