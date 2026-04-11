# Oracle RAC 19c 3 Node + ASM (AFD) — FULL FINAL HOW TO (RHEL 9)

---

## ENVIRONMENT

| Parameter       | Value                              |
| --------------- | ---------------------------------- |
| Cluster Name    | rac-cluster                        |
| Nodes           | plvmracdb1, plvmracdb2, plvmracdb3 |
| Node 1 IP       | 192.168.56.11                      |
| Node 2 IP       | 192.168.56.20                      |
| Node 3 IP       | 192.168.56.30                      |
| VIP             | 192.168.56.12,21,31                |
| SCAN            | rac-scan (192.168.56.90)           |
| Public Network  | 192.168.56.0/24                    |
| Private Network | 192.168.171.0/24                   |
| Grid Home       | /u01/app/19.0.0/grid               |
| DB Home         | /u01/app/oracle/product/19.0.0/db  |
| Oracle Base     | /u01/app/oracle                    |
| ASM Disk        | /dev/sdb, /dev/sdc                 |
| Diskgroup       | DATA                               |

---

# PART 0 — OS PREPARATION (ALL NODE)

```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config

systemctl stop firewalld
systemctl disable firewalld

dnf install -y chrony
systemctl enable chronyd
systemctl start chronyd

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

## GRID

```bash
su - grid
```

```bash
echo "export ORACLE_BASE=/u01/app/grid" >> ~/.bash_profile
echo "export ORACLE_HOME=/u01/app/19.0.0/grid" >> ~/.bash_profile
echo "export ORACLE_SID=+ASM1" >> ~/.bash_profile
echo "export PATH=\$ORACLE_HOME/bin:\$PATH" >> ~/.bash_profile
source ~/.bash_profile
```

---

## ORACLE

```bash
su - oracle
```

```bash
echo "export ORACLE_BASE=/u01/app/oracle" >> ~/.bash_profile
echo "export ORACLE_HOME=/u01/app/oracle/product/19.0.0/db" >> ~/.bash_profile
echo "export ORACLE_SID=RACDB1" >> ~/.bash_profile
echo "export PATH=\$ORACLE_HOME/bin:\$PATH" >> ~/.bash_profile
source ~/.bash_profile
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

# PART 4 — NETWORK (SESUAI ASLI)

## NODE 1 — plvmracdb1

```bash
hostnamectl set-hostname plvmracdb1

nmcli con mod enp0s8 ipv4.addresses 192.168.56.11/24
nmcli con mod enp0s8 ipv4.method manual

nmcli con mod enp0s9 ipv4.addresses 192.168.171.11/24
nmcli con mod enp0s9 ipv4.method manual

nmcli con up enp0s8
nmcli con up enp0s9
```

---

## NODE 2 — plvmracdb2

```bash
hostnamectl set-hostname plvmracdb2

nmcli con mod enp0s8 ipv4.addresses 192.168.56.20/24
nmcli con mod enp0s8 ipv4.method manual

nmcli con mod enp0s9 ipv4.addresses 192.168.171.20/24
nmcli con mod enp0s9 ipv4.method manual

nmcli con up enp0s8
nmcli con up enp0s9
```

---

## NODE 3 — plvmracdb3

```bash
hostnamectl set-hostname plvmracdb3

nmcli con mod enp0s8 ipv4.addresses 192.168.56.30/24
nmcli con mod enp0s8 ipv4.method manual

nmcli con mod enp0s9 ipv4.addresses 192.168.171.30/24
nmcli con mod enp0s9 ipv4.method manual

nmcli con up enp0s8
nmcli con up enp0s9
```

---

## /etc/hosts (ALL NODE)

```bash
127.0.0.1 localhost

192.168.56.11 plvmracdb1
192.168.56.20 plvmracdb2
192.168.56.30 plvmracdb3

192.168.56.12 plvmracdb1-vip
192.168.56.21 plvmracdb2-vip
192.168.56.31 plvmracdb3-vip

192.168.171.11 plvmracdb1-priv
192.168.171.20 plvmracdb2-priv
192.168.171.30 plvmracdb3-priv

192.168.56.90 rac-scan
```

---

# PART 5 — DISK PREPARATION

```bash
lsblk
```

```bash
chown grid:asmadmin /dev/sdb /dev/sdc
chmod 660 /dev/sdb /dev/sdc
```

---

# PART 6 — SSH SETUP (ALL USER)

## GRID

```bash
su - grid
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

for i in plvmracdb1 plvmracdb2 plvmracdb3
do
 ssh-copy-id grid@$i
done
```

---

## ORACLE

```bash
su - oracle
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

for i in plvmracdb1 plvmracdb2 plvmracdb3
do
 ssh-copy-id oracle@$i
done
```

---

## ROOT

```bash
sudo su -
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

for i in plvmracdb1 plvmracdb2 plvmracdb3
do
 ssh-copy-id root@$i
done
```

---

## TEST

```bash
ssh plvmracdb2 hostname
ssh plvmracdb3 hostname
```

---

# PART 7 — AFD CONFIG (SEBELUM GI)

```bash
su - grid
export ORACLE_HOME=/u01/app/19.0.0/grid
export PATH=$ORACLE_HOME/bin:$PATH

asmcmd afd_configure
asmcmd afd_state

asmcmd afd_label DATA1 /dev/sdb
asmcmd afd_label DATA2 /dev/sdc

asmcmd afd_lsdsk
```

---

# PART 8 — INSTALL GRID

```bash
su - grid
unzip LINUX.X64_193000_grid_home.zip -d /u01/app/19.0.0/grid
cd /u01/app/19.0.0/grid
./gridSetup.sh
```

SELECT:

* RAC
* ASM → AFD

---

# PART 9 — ROOT SCRIPT

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

---

# PART 10 — ASM DISKGROUP

* DATA
* External
* DATA1, DATA2

---

# PART 11 — VALIDATION

```bash
crsctl check cluster -all
crsctl stat res -t
olsnodes -n
```

---

# PART 12 — DATABASE

```bash
su - oracle
./runInstaller
dbca
```

---

# FINAL CHECK

```sql
select instance_name from gv$instance;
```

---

# RESULT

✔ RAC 3 Node Running
✔ ASM AFD OK
✔ Network sesuai original design
✔ SCAN & VIP aktif
✔ RHEL9 Ready

---
