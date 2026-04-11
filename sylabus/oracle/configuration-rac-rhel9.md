# Oracle RAC 19c 3 Node + ASM (AFD) — FULL HOW TO (RHEL 9)

---

## ENVIRONMENT

| Parameter       | Value                             |
| --------------- | --------------------------------- |
| Cluster Name    | rac-cluster                       |
| Nodes           | rac1, rac2, rac3                  |
| Node 1 IP       | 192.168.56.11                     |
| Node 2 IP       | 192.168.56.12                     |
| Node 3 IP       | 192.168.56.13                     |
| VIP             | 192.168.56.21-23                  |
| SCAN            | rac-scan (192.168.56.100)         |
| Public Network  | 192.168.56.0/24                   |
| Private Network | 192.168.171.0/24                  |
| Grid Home       | /u01/app/19.0.0/grid              |
| DB Home         | /u01/app/oracle/product/19.0.0/db |
| Oracle Base     | /u01/app/oracle                   |
| ASM Disk        | /dev/sdb, /dev/sdc                |
| Diskgroup       | DATA                              |

---

# PART 0 — OS PREPARATION

## 0.1 Disable SELinux

```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
```

## 0.2 Disable Firewall

```bash
systemctl stop firewalld
systemctl disable firewalld
```

## 0.3 Time Sync (MANDATORY RAC)

```bash
dnf install -y chrony
systemctl enable chronyd
systemctl start chronyd
chronyc sources
```

---

## 0.4 Install Required Packages

```bash
dnf install -y bc binutils elfutils-libelf gcc gcc-c++ glibc glibc-devel ksh \
libaio libaio-devel libXrender libXrender-devel libX11 libXau libXi \
libXtst make smartmontools sysstat unzip net-tools
```

---

# PART 1 — USER & GROUP

```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
groupadd -g 54324 asmadmin
groupadd -g 54325 asmdba
groupadd -g 54326 asmoper

useradd -u 54321 -g oinstall -G dba,asmdba,oper oracle
useradd -u 54322 -g oinstall -G asmadmin,asmdba,asmoper,dba grid

passwd oracle
passwd grid
```

---

# PART 2 — ENVIRONMENT VARIABLE

## GRID USER

```bash
su - grid
vi ~/.bash_profile
```

```bash
export ORACLE_BASE=/u01/app/grid
export ORACLE_HOME=/u01/app/19.0.0/grid
export ORACLE_SID=+ASM1
export PATH=$ORACLE_HOME/bin:$PATH
```

---

## ORACLE USER

```bash
su - oracle
vi ~/.bash_profile
```

```bash
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/db
export ORACLE_SID=RACDB1
export PATH=$ORACLE_HOME/bin:$PATH
```

---

# PART 3 — DIRECTORY

```bash
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/oracle/product/19.0.0/db
mkdir -p /u01/app/grid

chown -R grid:oinstall /u01/app
chmod -R 775 /u01/app
```

---

# PART 4 — NETWORK

## Interface Mapping

| Interface | Function |
| --------- | -------- |
| enp0s3    | NAT      |
| enp0s8    | PUBLIC   |
| enp0s9    | PRIVATE  |

---

## Configure IP (Each Node)

```bash
nmcli con mod enp0s8 ipv4.addresses 192.168.56.11/24
nmcli con mod enp0s8 ipv4.method manual
nmcli con mod enp0s9 ipv4.addresses 192.168.171.11/24
nmcli con mod enp0s9 ipv4.method manual
nmcli con up enp0s8
nmcli con up enp0s9
```

---

## /etc/hosts (ALL NODE)

```bash
127.0.0.1 localhost

# PUBLIC
192.168.56.11 rac1
192.168.56.12 rac2
192.168.56.13 rac3

# VIP
192.168.56.21 rac1-vip
192.168.56.22 rac2-vip
192.168.56.23 rac3-vip

# PRIVATE
192.168.171.11 rac1-priv
192.168.171.12 rac2-priv
192.168.171.13 rac3-priv

# SCAN
192.168.56.100 rac-scan
```

---

# PART 5 — DISK PREPARATION (AFD)

```bash
lsblk
```

Expected:

```
sdb
sdc
```

Set permission:

```bash
chown grid:asmadmin /dev/sdb /dev/sdc
chmod 660 /dev/sdb /dev/sdc
```

---

# PART 6 — SSH SETUP

```bash
su - grid
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

for i in rac1 rac2 rac3
do
 ssh-copy-id $i
done
```

Test:

```bash
ssh rac2 hostname
```

---

# PART 7 — INSTALL GRID

```bash
su - grid
unzip LINUX.X64_193000_grid_home.zip -d /u01/app/19.0.0/grid
cd /u01/app/19.0.0/grid
./gridSetup.sh
```

---

## Installer Setting

* Cluster Name : rac-cluster
* SCAN         : rac-scan
* Network      :

  * Public  → enp0s8
  * Private → enp0s9

⚠️ IMPORTANT:

* SELECT: ASM Filter Driver (AFD)
* DO NOT SELECT: ASMLib

---

# PART 8 — AFD CONFIG

```bash
su - grid
asmcmd afd_configure
asmcmd afd_state
```

---

## Label Disk

```bash
asmcmd afd_label DATA1 /dev/sdb
asmcmd afd_label DATA2 /dev/sdc
```

---

## Verify

```bash
asmcmd afd_lsdsk
```

---

# PART 9 — ROOT SCRIPT

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

---

# PART 10 — ASM DISKGROUP

* Name        : DATA
* Redundancy  : External
* Disk        : DATA1, DATA2

---

# PART 11 — VALIDATION

```bash
crsctl check cluster -all
crsctl stat res -t
olsnodes -n
```

---

# PART 12 — DATABASE INSTALL

```bash
su - oracle
./runInstaller
dbca
```

---

# PART 13 — FINAL VALIDATION

```sql
select instance_name from gv$instance;
```

---

# SUMMARY

✔ RAC 3 Node Running
✔ ASM AFD (RHEL9 compatible)
✔ No ASMLib
✔ Cluster Healthy
✔ Production Ready Lab

---
