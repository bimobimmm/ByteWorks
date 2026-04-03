# Oracle RAC 19c Installation (3 Node - ByteWorks Full Guide)

---

# 1. Environment

| Node | Hostname | Public IP |
|------|----------|----------|
| Node1 | PLVMRACDB1 | 192.168.56.10 |
| Node2 | PLVMRACDB2 | 192.168.56.20 |
| Node3 | PLVMRACDB3 | 192.168.56.30 |

---

# 2. Software Download

Download:
https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html

File:
- LINUX.X64_193000_grid_home.zip  
- LINUX.X64_193000_db_home.zip  

---

# 3. VM Configuration

## Spec per node

- RAM: 3–5 GB  
- CPU: 1–2 Core  
- Disk: 25–40 GB  

---

## Network Adapter

- Adapter 1 → NAT  
- Adapter 2 → Host-Only  

---

# 4. Create Shared ASM Disk (HOST)

```bash
cd "C:\Program Files\Oracle\VirtualBox"

VBoxManage createmedium disk --filename D:\RAC\asm1.vdi --size 8192 --format VDI
VBoxManage createmedium disk --filename D:\RAC\asm2.vdi --size 8192 --format VDI

VBoxManage modifyhd D:\RAC\asm1.vdi --type shareable
VBoxManage modifyhd D:\RAC\asm2.vdi --type shareable
```

---

# 5. Attach Disk ke Semua VM

Tambahkan:
- asm1.vdi → /dev/sdb  
- asm2.vdi → /dev/sdc  

---

# 6. OS Preparation (ALL NODE)

```bash
hostnamectl set-hostname PLVMRACDB1   # sesuaikan tiap node

systemctl stop firewalld
systemctl disable firewalld

setenforce 0

yum install -y oracle-database-preinstall-19c
yum install -y oracleasm-support oracleasm
```

---

# 7. Directory Setup

```bash
mkdir -p /u01/app/oracle
mkdir -p /u01/app/oraInventory
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1

chown -R oracle:oinstall /u01
chmod -R 775 /u01
```

---

# 8. Network Configuration

## Set IP

### Node1

```bash
nmcli con mod eth0 ipv4.addresses 192.168.56.10/24 ipv4.method manual
nmcli con mod eth1 ipv4.addresses 10.10.10.1/24 ipv4.method manual
nmcli con up eth0
nmcli con up eth1
```

---

### Node2

```bash
nmcli con mod eth0 ipv4.addresses 192.168.56.20/24 ipv4.method manual
nmcli con mod eth1 ipv4.addresses 10.10.10.2/24 ipv4.method manual
```

---

### Node3

```bash
nmcli con mod eth0 ipv4.addresses 192.168.56.30/24 ipv4.method manual
nmcli con mod eth1 ipv4.addresses 10.10.10.3/24 ipv4.method manual
```

---

## /etc/hosts (ALL NODE)

```bash
192.168.56.10 PLVMRACDB1
192.168.56.20 PLVMRACDB2
192.168.56.30 PLVMRACDB3

192.168.56.11 PLVMRACDB1-vip
192.168.56.21 PLVMRACDB2-vip
192.168.56.31 PLVMRACDB3-vip

10.10.10.1 PLVMRACDB1-priv
10.10.10.2 PLVMRACDB2-priv
10.10.10.3 PLVMRACDB3-priv

192.168.56.100 rac-scan
```

---

## Validation Network

```bash
ping PLVMRACDB2
ping PLVMRACDB3
ping rac-scan
```

---

# 9. SSH Passwordless

```bash
ssh-keygen -t rsa
ssh-copy-id PLVMRACDB1
ssh-copy-id PLVMRACDB2
ssh-copy-id PLVMRACDB3
```

---

# 10. ASM Configuration (SUPER DETAIL)

## Verify Disk

```bash
lsblk
```

---

## Configure ASM

```bash
oracleasm configure -i
```

---

## Init ASM

```bash
oracleasm init
```

---

## Create Disk (NODE1 ONLY)

```bash
oracleasm createdisk DATA1 /dev/sdb
oracleasm createdisk DATA2 /dev/sdc
```

---

## Scan Disk (ALL NODE)

```bash
oracleasm scandisks
```

---

## Verify

```bash
oracleasm listdisks
ls -l /dev/oracleasm/disks/
```

---

# 11. Install Grid Infrastructure

```bash
cd /u01/app/19.0.0/grid
./gridSetup.sh
```

---

## Config:

- Cluster Name: BYTEWORK_CLUSTER  
- SCAN Name: rac-scan  
- SCAN Port: 1521  

Nodes:
- PLVMRACDB1  
- PLVMRACDB2  
- PLVMRACDB3  

---

## Network

- eth0 → PUBLIC  
- eth1 → PRIVATE  

---

## ASM Disk

- DATA1  
- DATA2  

---

## Root Script

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

---

# 12. Install Database (DBCA)

```bash
dbca
```

---

## Config

- RAC Database  
- DB Name: BYTEWORK  
- Nodes: 3  
- Storage: ASM  

---

# 13. Listener (AUTO)

```bash
srvctl status listener
srvctl status scan_listener

srvctl start listener
srvctl start scan_listener
```

---

# 14. TNS Configuration

```bash
vi $ORACLE_HOME/network/admin/tnsnames.ora
```

---

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

## Test

```bash
tnsping BYTEWORK
```

---

# 15. Final Validation

```bash
crsctl status resource -t
olsnodes -n
srvctl status database -d BYTEWORK
srvctl status listener
srvctl status scan_listener
srvctl status scan
```

---

# 16. Connection Flow

Client → rac-scan → scan listener → node → instance

---

# SUMMARY

- Network: public, private, VIP, SCAN  
- ASM disk dari VirtualBox (shared)  
- ASM dibuat di node1  
- Grid menggunakan ASM  
- Listener otomatis  
- TNS menggunakan SCAN  
- RAC berjalan multi-node  
