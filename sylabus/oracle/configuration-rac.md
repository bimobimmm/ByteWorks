# Oracle RAC 19c 3 Node + ASM Configuration Guide (ByteW@rks)

---

## Environment

| Parameter       | Value                              |
| --------------- | ---------------------------------- |
| Cluster Name    | rac-cluster                        |
| Nodes           | plvmracdb1, plvmracdb2, plvmracdb3 |
| Node 1 IP       | 192.168.56.11                      |
| Node 2 IP       | 192.168.56.20                      |
| Node 3 IP       | 192.168.56.30                      |
| Grid Home       | /u01/app/19.0.0/grid               |
| Oracle Base     | /u01/app/grid                      |
| Public Network  | 192.168.56.0/24                    |
| Private Network | 192.168.171.0/24                   |
| SCAN            | rac-scan (192.168.56.90)           |
| ASM Diskgroup   | DATA                               |
| ASM Disk        | DATA1, DATA2                       |

---

# PART 1 — ASM SETUP (VIRTUALBOX)

## 1. Create ASM Disk (Host)

```bash
VBoxManage createmedium disk --filename D:\RAC\asm1.vdi --size 8192 --format VDI --variant Fixed
VBoxManage createmedium disk --filename D:\RAC\asm2.vdi --size 8192 --format VDI --variant Fixed

VBoxManage modifyhd D:\RAC\asm1.vdi --type shareable
VBoxManage modifyhd D:\RAC\asm2.vdi --type shareable
```

---

## 2. Attach Disk ke Semua Node

* VirtualBox → Settings → Storage
* Tambahkan:

  * asm1.vdi
  * asm2.vdi
* Mode: **SHAREABLE**

---

## 3. Verifikasi Disk

```bash
lsblk
```

Expected:

```bash
sdb 8G
sdc 8G
```

---

# PART 2 — ASM CONFIGURATION

## 4. Install ASM Package

```bash
yum install -y oracleasm-support oracleasmlib oracleasm
```

---

## 5. Configure ASM

```bash
oracleasm configure -i
```

Input:

```
oracle
oinstall
y
y
```

---

## 6. Initialize ASM

```bash
oracleasm init
```

---

## 7. Partition Disk

```bash
fdisk /dev/sdb
fdisk /dev/sdc
```

Steps:

```
n → p → 1 → enter → enter → w
```

---

## 8. Create ASM Disk

```bash
oracleasm createdisk DATA1 /dev/sdb1
oracleasm createdisk DATA2 /dev/sdc1
```

---

## 9. Scan & Verify

```bash
oracleasm scandisks
oracleasm listdisks
ls -l /dev/oracleasm/disks/
```

---

# PART 3 — NETWORK CONFIGURATION

## 10. Adapter Mapping

| Interface | Function |
| --------- | -------- |
| enp0s3    | NAT      |
| enp0s8    | PUBLIC   |
| enp0s9    | PRIVATE  |

---

## 11. Configure IP (SETIAP NODE BERBEDA)

### Node 1 (plvmracdb1)

```bash
nmcli con mod PUBLIC ipv4.addresses 192.168.56.11/24
nmcli con mod PUBLIC ipv4.method manual
nmcli con mod PRIVATE ipv4.addresses 192.168.171.11/24
nmcli con mod PRIVATE ipv4.method manual
nmcli con up PUBLIC
nmcli con up PRIVATE
```

### Node 2 (plvmracdb2)

```bash
nmcli con mod PUBLIC ipv4.addresses 192.168.56.20/24
nmcli con mod PUBLIC ipv4.method manual
nmcli con mod PRIVATE ipv4.addresses 192.168.171.20/24
nmcli con mod PRIVATE ipv4.method manual
nmcli con up PUBLIC
nmcli con up PRIVATE
```

### Node 3 (plvmracdb3)

```bash
nmcli con mod PUBLIC ipv4.addresses 192.168.56.30/24
nmcli con mod PUBLIC ipv4.method manual
nmcli con mod PRIVATE ipv4.addresses 192.168.171.30/24
nmcli con mod PRIVATE ipv4.method manual
nmcli con up PUBLIC
nmcli con up PRIVATE
```

---

## 12. /etc/hosts (SEMUA NODE HARUS SAMA)

```bash
127.0.0.1 localhost

# PUBLIC
192.168.56.11 plvmracdb1
192.168.56.20 plvmracdb2
192.168.56.30 plvmracdb3

# VIP
192.168.56.12 plvmracdb1-vip
192.168.56.21 plvmracdb2-vip
192.168.56.31 plvmracdb3-vip

# PRIVATE
192.168.171.11 plvmracdb1-priv
192.168.171.20 plvmracdb2-priv
192.168.171.30 plvmracdb3-priv

# SCAN
192.168.56.90 rac-scan
```

---

# PART 4 — USER & DIRECTORY

## 13. Create Group

```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54331 asmadmin
groupadd -g 54332 asmdba
groupadd -g 54333 asmoper
```

---

## 14. Create User

```bash
useradd -u 1001 -g oinstall -G dba,asmadmin,asmdba,asmoper grid
useradd -u 1002 -g oinstall -G dba,asmdba oracle

passwd grid
passwd oracle
```

---

## 15. Directory Structure

```bash
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/grid
mkdir -p /u01/app/oracle

chown -R grid:oinstall /u01/app
chmod -R 775 /u01/app
```

---

# PART 5 — OS CONFIGURATION

## 16. limits.conf

```bash
vi /etc/security/limits.conf

grid hard nofile 65536
grid hard nproc 16384
grid soft stack 10240
```

---

## 17. sysctl.conf

```bash
vi /etc/sysctl.conf

fs.file-max = 6815744
kernel.sem = 250 32000 100 128
```

Apply:

```bash
sysctl -p
```

---

# PART 6 — SSH SETUP (WAJIB)

## 18. GRID USER

```bash
su - grid
ssh-keygen
ssh-copy-id grid@plvmracdb1
ssh-copy-id grid@plvmracdb2
ssh-copy-id grid@plvmracdb3
```

---

## 19. ORACLE USER

```bash
su - oracle
ssh-keygen
ssh-copy-id oracle@plvmracdb1
ssh-copy-id oracle@plvmracdb2
ssh-copy-id oracle@plvmracdb3
```

---

## 20. ROOT USER

```bash
sudo su -
ssh-keygen
ssh-copy-id root@plvmracdb1
ssh-copy-id root@plvmracdb2
ssh-copy-id root@plvmracdb3
```

---

## TEST SSH

```bash
ssh plvmracdb2 hostname
ssh plvmracdb3 hostname
```

---

# PART 7 — GRID INSTALLATION

## 21. Extract Grid

```bash
su - grid
unzip grid_home.zip -d /u01/app/19.0.0/grid
```

---

## 22. Run Installer

```bash
cd /u01/app/19.0.0/grid
./gridSetup.sh
```

---

## 23. Configuration

* Cluster Name : rac-cluster
* SCAN Name    : rac-scan

Network:

* Public  → enp0s8
* Private → enp0s9

---

## 24. Root Script

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

---

# PART 8 — ASM DISKGROUP

## 25. Create Diskgroup

* Name        : DATA
* Disk        : DATA1, DATA2
* Redundancy  : External

---

# PART 9 — VALIDATION GRID

```bash
crsctl check cluster -all
crsctl stat res -t
olsnodes -n
```

---

# PART 10 — DATABASE INSTALLATION

## 26. Install Oracle Database

```bash
su - oracle
./runInstaller
```

---

## 27. Create RAC Database

```bash
dbca
```

---

# PART 11 — FINAL VALIDATION

```sql
sqlplus / as sysdba
select instance_name from gv$instance;
```

---

# SUMMARY

* RAC 3 Node running
* ASM Diskgroup mounted
* SCAN listener aktif
* VIP aktif
* Cluster healthy
