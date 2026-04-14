# Oracle RAC 19c 2 Node + ASM Configuration Guide (ByteWorks)

---

## Environment

| Parameter     | Value                                   |
| ------------- | --------------------------------------- |
| Nodes         | plvmracdb1, plvmracdb2                  |
| Public IP     | 192.168.56.x                            |
| Private IP    | 192.168.171.x                           |
| Grid Home     | /u01/app/19.0.0/grid                    |
| DB Home       | /u01/app/oracle/product/19.0.0/dbhome_1 |
| ASM Diskgroup | DATA                                    |

---

# PART 1 — OS PREPARATION

## 1. Set Hostname

📍 All Node | root
Menentukan identitas node dalam cluster.

```bash
hostnamectl set-hostname plvmracdb1
hostnamectl set-hostname plvmracdb2
```

---

## 2. Disable Firewall & SELinux

📍 All Node | root
Menghindari blocking komunikasi RAC.

```bash
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
```

---

## 3. Install Preinstall Package

📍 All Node | root
Menginstall dependency Oracle otomatis.

```bash
yum install -y oracle-database-preinstall-19c
```

---

# PART 2 — NETWORK CONFIG

## 4. Configure Network

📍 All Node | root
Mengatur IP public & private.

### Node1

```bash
nmcli con mod enp0s8 ipv4.addresses 192.168.56.11/24
nmcli con mod enp0s8 ipv4.method manual

nmcli con mod enp0s9 ipv4.addresses 192.168.171.11/24
nmcli con mod enp0s9 ipv4.method manual
```

### Node2

```bash
nmcli con mod enp0s8 ipv4.addresses 192.168.56.20/24
nmcli con mod enp0s8 ipv4.method manual

nmcli con mod enp0s9 ipv4.addresses 192.168.171.20/24
nmcli con mod enp0s9 ipv4.method manual
```

---

## 5. Auto UP Network (PENTING)

📍 All Node | root
Agar interface aktif saat reboot.

```bash
nmcli con mod enp0s8 connection.autoconnect yes
nmcli con mod enp0s9 connection.autoconnect yes
```

---

## 6. /etc/hosts

📍 All Node | root
Resolusi hostname cluster.

```bash
192.168.56.11 plvmracdb1
192.168.56.20 plvmracdb2

192.168.56.12 plvmracdb1-vip
192.168.56.21 plvmracdb2-vip

192.168.171.11 plvmracdb1-priv
192.168.171.20 plvmracdb2-priv

192.168.56.90 rac-scan
```

---

# PART 3 — USER & DIRECTORY

## 7. Create User & Group

📍 All Node | root
User harus identik antar node.

```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54331 asmadmin
groupadd -g 54332 asmdba
groupadd -g 54333 asmoper

useradd -u 1001 -g oinstall -G dba,asmadmin,asmdba,asmoper grid
useradd -u 1002 -g oinstall -G dba,asmdba oracle
```

---

## 8. Directory

📍 All Node | root
Struktur Oracle wajib konsisten.

```bash
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/grid
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1

chown -R grid:oinstall /u01/app
chmod -R 775 /u01/app
```

---

# PART 4 — ASM SETUP

## 9. Install ASM Package

📍 All Node | root
Dependency ASM.

```bash
yum install -y oracleasm-support oracleasmlib oracleasm
```

---

## 10. Configure ASM

📍 All Node | root
Menentukan owner ASM.

```bash
oracleasm configure -i
```

Isi:

```
Default user: grid
Default group: oinstall
Start on boot: y
Scan on boot: y
```

---

## 11. Initialize ASM

📍 All Node | root
Menjalankan ASM service.

```bash
oracleasm init
```

---

## 12. Partition Disk (DETAIL)

📍 All Node | root
Membuat partisi ASM.

```bash
fdisk /dev/sdb
```

Step:

```
n → p → 1 → ENTER → ENTER → w
```

Ulangi untuk `/dev/sdc`

---

## 13. Create ASM Disk

📍 HANYA NODE1 | root
Register disk ke ASM.

```bash
oracleasm createdisk DATA1 /dev/sdb1
oracleasm createdisk DATA2 /dev/sdc1
```

---

## 14. Scan ASM

📍 ALL NODE | root
Sinkronisasi disk.

```bash
oracleasm scandisks
oracleasm listdisks
```

👉 Node2 akan otomatis detect disk dari node1

---

# PART 5 — SSH SETUP

## 15. Passwordless SSH

📍 Node1

```bash
su - grid
ssh-keygen
ssh-copy-id grid@plvmracdb1
ssh-copy-id grid@plvmracdb2

su - oracle
ssh-keygen
ssh-copy-id oracle@plvmracdb1
ssh-copy-id oracle@plvmracdb2
```

---

# PART 6 — GRID INSTALLATION

## 16. Unzip Grid

📍 Node1 | grid

```bash
unzip grid_home.zip -d /u01/app/19.0.0/grid
```

---

## 17. Run Installer

```bash
./gridSetup.sh
```

Pilih:

* RAC
* ASM
* Diskgroup DATA

---

## 18. Root Script

📍 All Node | root

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

---

# PART 7 — VALIDATION

```bash
crsctl check cluster -all
crsctl stat res -t
olsnodes -n
```

---

# PART 8 — DATABASE INSTALLATION

## 19. Unzip DB Home

📍 Node1 | oracle

```bash
unzip db_home.zip -d /u01/app/oracle/product/19.0.0/dbhome_1
```

---

## 20. Set Permission

📍 All Node | root

```bash
chown -R oracle:oinstall /u01/app/oracle
chmod -R 775 /u01/app/oracle
```

---

## 21. Install DB

📍 Node1 | oracle

```bash
./runInstaller
```

---

## 22. Create Database

```bash
dbca
```

---

# PART 9 — VALIDATION

```sql
select instance_name, host_name from gv$instance;
```

---

# SUMMARY

Environment berhasil dikonfigurasi dari nol hingga database RAC berjalan dengan kondisi berikut:

- Network PUBLIC (enp0s8) dan PRIVATE (enp0s9) telah dikonfigurasi secara manual menggunakan nmcli
- Interface PRIVATE (enp0s9) sudah diset autoconnect sehingga otomatis UP saat reboot
- Resolusi hostname menggunakan /etc/hosts untuk PUBLIC, VIP, PRIVATE, dan SCAN berhasil dikonfigurasi konsisten di semua node
- Package preinstall Oracle 19c berhasil diinstall untuk memenuhi dependency OS

- User dan group (grid, oracle, oinstall, dba, asm*) telah dibuat identik di semua node
- Struktur direktori Oracle (/u01/app) termasuk GRID HOME dan DB HOME telah dibuat dan diberikan ownership yang sesuai

- ASM berhasil dikonfigurasi menggunakan oracleasm dengan user grid dan group oinstall
- Disk ASM dibuat melalui proses partisi (fdisk) dan hanya diregistrasi di node1
- Node2 berhasil mendeteksi disk ASM melalui proses scan (oracleasm scandisks)
- Diskgroup DATA berhasil dibuat dan dapat diakses oleh kedua node

- Passwordless SSH berhasil dikonfigurasi untuk user grid dan oracle antar node

- Grid Infrastructure berhasil diinstall dan cluster terbentuk dengan 2 node
- Semua komponen cluster (CRS, CSS, ASM, Listener, VIP, SCAN) dalam kondisi ONLINE dan STABLE
- Validasi cluster menggunakan crsctl dan olsnodes menunjukkan hasil normal

- Oracle Database software berhasil diinstall menggunakan user oracle
- Database RAC berhasil dibuat menggunakan DBCA dengan storage ASM (diskgroup DATA)
- Instance database berjalan di kedua node (byteworks1 dan byteworks2)

- Environment oracle telah dikonfigurasi melalui .bash_profile dengan SID otomatis per node
- Cluster dapat dikontrol menggunakan srvctl dan seluruh resource berjalan normal

- RAC Backup Script berhasil dibuat untuk mendokumentasikan konfigurasi:
  - Mengambil data system, network, cluster, ASM, listener
  - Backup bash_profile user grid dan oracle
  - Output dalam format TXT dan HTML (dark UI)
  - File dinamai berdasarkan hostname masing-masing node

---
