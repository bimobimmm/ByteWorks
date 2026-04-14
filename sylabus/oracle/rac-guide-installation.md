# Oracle RAC 19c 2 Node + ASM Configuration Guide (ByteWorks)

---

## Topology

```
                +----------------------+
                |      rac-scan        |
                |   192.168.56.90      |
                +----------+-----------+
                           |
        -----------------------------------------
        |                                       |
+-------------------+                +-------------------+
|   plvmracdb1      |                |   plvmracdb2      |
|-------------------|                |-------------------|
| Public  : .11     |                | Public  : .20     |
| VIP     : .12     |                | VIP     : .21     |
| Private : .11     |                | Private : .20     |
| ASM     : +ASM1   |                | ASM     : +ASM2   |
| DB      : bw1     |                | DB      : bw2     |
+-------------------+                +-------------------+
        |                                       |
        ----------- Shared ASM Disk -------------
                  (sdb, sdc)
```

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

# PART 0 — ASM DISK (HOST / VIRTUALBOX)

## 0.1 Create ASM Disk

📍 Host (CMD / PowerShell)
Membuat disk virtual yang akan digunakan sebagai shared storage ASM.

```bash
VBoxManage createmedium disk --filename D:\RAC\asm1.vdi --size 8192 --format VDI --variant Fixed
VBoxManage createmedium disk --filename D:\RAC\asm2.vdi --size 8192 --format VDI --variant Fixed
```

---

## 0.2 Set Disk Shareable

📍 Host
Agar disk bisa digunakan oleh lebih dari 1 VM (requirement RAC).

```bash
VBoxManage modifyhd D:\RAC\asm1.vdi --type shareable
VBoxManage modifyhd D:\RAC\asm2.vdi --type shareable
```

---

## 0.3 Attach Disk ke VM

📍 VirtualBox GUI

Tambahkan ke **SEMUA NODE**:

* asm1.vdi
* asm2.vdi

Mode:

* Shareable

---

## 0.4 Verifikasi Disk

📍 All Node | root
Memastikan disk sudah terbaca OS.

```bash
lsblk
```

Expected:

```
sdb
sdc
```

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

* Network RAC berhasil dikonfigurasi (public & private, auto up)
* ASM shared disk berhasil dibuat dan digunakan oleh kedua node
* Disk ASM dibuat di node1 dan otomatis terdeteksi di node2
* Grid Infrastructure terinstall dan cluster dalam kondisi ONLINE
* Semua resource cluster berjalan normal
* Database RAC berhasil dibuat dan berjalan di kedua node
* Instance database aktif di masing-masing node

```
