# Oracle RAC 19c Installation Guide (3 Node - ByteWorks)

---

## 1. Environment

| Node | Hostname | Public IP |
|------|----------|----------|
| Node1 | PLVMRACDB1 | 192.168.56.10 |
| Node2 | PLVMRACDB2 | 192.168.56.20 |
| Node3 | PLVMRACDB3 | 192.168.56.30 |

---

## 2. Software Download

Download dari Oracle:

https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html  

**File:**
- LINUX.X64_193000_grid_home.zip  
- LINUX.X64_193000_db_home.zip  

---

## 3. VM Configuration

**Per Node:**
- RAM: 3–5 GB  
- CPU: 1–2 Core  
- Disk: 25–40 GB  

**Network Adapter:**
- Adapter 1 → NAT  
- Adapter 2 → Host-Only  

---

## 4. Create Shared ASM Disk (HOST)

```bash
cd "C:\Program Files\Oracle\VirtualBox"

VBoxManage createmedium disk --filename D:\RAC\asm1.vdi --size 8192 --format VDI
VBoxManage createmedium disk --filename D:\RAC\asm2.vdi --size 8192 --format VDI

VBoxManage modifyhd D:\RAC\asm1.vdi --type shareable
VBoxManage modifyhd D:\RAC\asm2.vdi --type shareable
```

---

## 5. Attach Disk ke Semua VM

Tambahkan disk berikut ke:
- PLVMRACDB1  
- PLVMRACDB2  
- PLVMRACDB3  

---

## 6. OS Preparation (ALL NODE)

```bash
hostnamectl set-hostname PLVMRACDB1   # sesuaikan tiap node

systemctl stop firewalld
systemctl disable firewalld

setenforce 0

yum install -y oracle-database-preinstall-19c
yum install -y oracleasm-support oracleasm
```

---

## 7. Directory Setup

```bash
mkdir -p /u01/app/oracle
mkdir -p /u01/app/oraInventory
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1

chown -R oracle:oinstall /u01
chmod -R 775 /u01
```

---

## 8. Network Configuration

### Set IP

#### Node1
```bash
nmcli con mod eth0 ipv4.addresses 192.168.56.10/24 ipv4.method manual
nmcli con mod eth1 ipv4.addresses 10.10.10.1/24 ipv4.method manual
nmcli con up eth0
nmcli con up eth1
```

#### Node2
```bash
nmcli con mod eth0 ipv4.addresses 192.168.56.20/24 ipv4.method manual
nmcli con mod eth1 ipv4.addresses 10.10.10.2/24 ipv4.method manual
```

#### Node3
```bash
nmcli con mod eth0 ipv4.addresses 192.168.56.30/24 ipv4.method manual
nmcli con mod eth1 ipv4.addresses 10.10.10.3/24 ipv4.method manual
```

---

### /etc/hosts (WAJIB SAMA DI SEMUA NODE)

```bash
# PUBLIC
192.168.56.10 PLVMRACDB1
192.168.56.20 PLVMRACDB2
192.168.56.30 PLVMRACDB3

# VIP
192.168.56.11 PLVMRACDB1-vip
192.168.56.21 PLVMRACDB2-vip
192.168.56.31 PLVMRACDB3-vip

# PRIVATE
10.10.10.1 PLVMRACDB1-priv
10.10.10.2 PLVMRACDB2-priv
10.10.10.3 PLVMRACDB3-priv

# SCAN
192.168.56.100 rac-scan
```

---

### Validation

```bash
ping PLVMRACDB2
ping PLVMRACDB3
ping rac-scan
```

---

## 9. SSH Passwordless

```bash
ssh-keygen -t rsa
ssh-copy-id PLVMRACDB1
ssh-copy-id PLVMRACDB2
ssh-copy-id PLVMRACDB3
```

---

## 10. ASM Configuration (DETAIL)

### Verify Disk

```bash
lsblk
```

Expected:
- sdb  
- sdc  

---

### Configure ASM (ALL NODE)

```bash
oracleasm configure -i
```

---

### Initialize

```bash
oracleasm init
```

---

### Create Disk (ONLY NODE1)

```bash
oracleasm createdisk DATA1 /dev/sdb
oracleasm createdisk DATA2 /dev/sdc
```

---

### Scan Disk (ALL NODE)

```bash
oracleasm scandisks
oracleasm listdisks
```

---

### Verify

```bash
ls -l /dev/oracleasm/disks/
```

---

## 11. Grid Installation

```bash
cd /u01/app/19.0.0/grid
./gridSetup.sh
```

---

### Config:

- Cluster Name: BYTEWORK_CLUSTER  
- SCAN: rac-scan  
- Nodes:
  - PLVMRACDB1  
  - PLVMRACDB2  
  - PLVMRACDB3  

---

### Network:

- eth0 → PUBLIC  
- eth1 → PRIVATE  

---

### ASM:

- DATA1  
- DATA2  

---

### Root Script

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

---

## 12. Database Installation

```bash
dbca
```

---

### Config:

- RAC Database  
- DB Name: BYTEWORK  
- Nodes: 3  
- Storage: ASM  

---

## 13. Listener

```bash
srvctl status listener
srvctl status scan_listener

srvctl start listener
srvctl start scan_listener
```

---

## 14. TNS Configuration

```bash
vi $ORACLE_HOME/network/admin/tnsnames.ora
```

```ini
BYTEWORK =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = rac-scan)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = BYTEWORK)
    )
  )
```

---

### Test

```bash
tnsping BYTEWORK
```

---

## 15. Validation

```bash
crsctl status resource -t
olsnodes -n
srvctl status database -d BYTEWORK
srvctl status listener
srvctl status scan_listener
```

---

## 16. Connection Flow

Client → SCAN → SCAN Listener → Node → Instance

---

# Summary

- RAC terdiri dari 3 node  
- ASM disk shared dari VirtualBox  
- Network terdiri dari public, private, VIP, SCAN  
- Listener otomatis oleh Grid Infrastructure  
- TNS menggunakan SCAN  
- Cluster siap digunakan untuk HA  
