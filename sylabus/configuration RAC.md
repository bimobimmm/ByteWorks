# Oracle RAC 19c Installation (3 Node - ByteWorks Complete Guide)

---

# 1. Environment

| Node | Hostname | Public IP |
|------|----------|----------|
| Node1 | PLVMRACDB1 | 192.168.56.10 |
| Node2 | PLVMRACDB2 | 192.168.56.20 |
| Node3 | PLVMRACDB3 | 192.168.56.30 |

Private Network:
- 10.10.10.x

SCAN:
- rac-scan → 192.168.56.100

---

# 2. VM Configuration (VirtualBox)

Per VM:
- RAM: 3–5 GB
- CPU: 1–2 Core
- Disk: 25–40 GB

Network:
- Adapter 1 → NAT
- Adapter 2 → Host-Only

---

# 3. Create Shared ASM Disk (HOST)

```bash
cd "C:\Program Files\Oracle\VirtualBox"

VBoxManage createmedium disk --filename D:\RAC\asm1.vdi --size 8192 --format VDI
VBoxManage createmedium disk --filename D:\RAC\asm2.vdi --size 8192 --format VDI

VBoxManage modifyhd D:\RAC\asm1.vdi --type shareable
VBoxManage modifyhd D:\RAC\asm2.vdi --type shareable
```

Attach ke semua VM:
- asm1 → /dev/sdb
- asm2 → /dev/sdc

---

# 4. OS Preparation (ALL NODE)

```bash
hostnamectl set-hostname PLVMRACDB1   # sesuaikan tiap node

systemctl stop firewalld
systemctl disable firewalld

setenforce 0

yum install -y oracle-database-preinstall-19c
yum install -y oracleasm-support oracleasm
```

---

# 5. Directory Setup

```bash
mkdir -p /u01/app/oracle
mkdir -p /u01/app/oraInventory
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1

chown -R oracle:oinstall /u01
chmod -R 775 /u01
```

---

# 6. Network Configuration

## Node1

```bash
nmcli con mod eth0 ipv4.addresses 192.168.56.10/24 ipv4.method manual
nmcli con mod eth1 ipv4.addresses 10.10.10.1/24 ipv4.method manual
nmcli con up eth0
nmcli con up eth1
```

## Node2

```bash
nmcli con mod eth0 ipv4.addresses 192.168.56.20/24 ipv4.method manual
nmcli con mod eth1 ipv4.addresses 10.10.10.2/24 ipv4.method manual
```

## Node3

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

## Network Validation

```bash
ping PLVMRACDB2
ping PLVMRACDB3
ping rac-scan
```

---

# 7. SSH Passwordless

```bash
ssh-keygen -t rsa
ssh-copy-id PLVMRACDB1
ssh-copy-id PLVMRACDB2
ssh-copy-id PLVMRACDB3
```

---

# 8. ASM Configuration

## Check Disk

```bash
lsblk
```

Expected:
- sdb
- sdc

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

# 9. Extract Software

## Grid

```bash
unzip LINUX.X64_193000_grid_home.zip -d /u01/app/19.0.0/grid
```

## DB

```bash
unzip LINUX.X64_193000_db_home.zip -d /u01/app/oracle/product/19.0.0/dbhome_1
```

---

# 10. Install Grid Infrastructure

```bash
cd /u01/app/19.0.0/grid
./gridSetup.sh
```

---

## Pilih:

- Cluster Name: BYTEWORK_CLUSTER
- SCAN Name: rac-scan
- Nodes: 3

---

## Network:

- eth0 → PUBLIC
- eth1 → PRIVATE

---

## ASM Disk:

- DATA1
- DATA2

---

## Run Root Script

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

---

# 11. Install Database

```bash
dbca
```

---

## Config:

- RAC Database
- DB Name: BYTEWORK
- Storage: ASM
- Nodes: 3

---

# 12. Listener

```bash
srvctl status listener
srvctl status scan_listener

srvctl start listener
srvctl start scan_listener
```

---

# 13. TNS

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

# 14. Final Validation

```bash
crsctl status resource -t
olsnodes -n
srvctl status database -d BYTEWORK
srvctl status scan
srvctl status scan_listener
```

---

# SUMMARY

- RAC 3 Node berjalan
- ASM menggunakan shared disk
- SCAN sebagai entry point
- Listener otomatis dari Grid
- Database berjalan multi-instance
