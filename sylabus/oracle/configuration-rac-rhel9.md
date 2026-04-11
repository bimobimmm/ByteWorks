# Oracle RAC 19c 3 Node + ASM (AFD) — FULL FINAL HOW TO (RHEL 9)

---

## ENVIRONMENT

| Parameter       | Value                              |
| --------------- | ---------------------------------- |
| Cluster Name    | rac-cluster                        |
| Nodes           | plvmracdb1, plvmracdb2,            |
| Node 1 IP       | 192.168.56.11                      |
| Node 2 IP       | 192.168.56.20                      |
| VIP             | 192.168.56.12,21,                  |
| SCAN            | rac-scan (192.168.56.90)           |
| Public Network  | 192.168.56.0/24                    |
| Private Network | 192.168.171.0/24                   |
| Grid Home       | /u01/app/19.0.0/grid               |
| Oracle Base     | /u01/app/oracle                    |
| Diskgroup       | DATA                               |

---

# PART 0 — OS PREPARATION

Menyiapkan OS agar tidak menghalangi instalasi Oracle serta memenuhi dependency.

```bash id="p0"
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config

systemctl stop firewalld
systemctl disable firewalld

dnf install -y chrony
systemctl enable chronyd
systemctl start chronyd

dnf install -y bc binutils elfutils-libelf gcc gcc-c++ glibc glibc-devel \
ksh libaio libaio-devel libXrender libX11 libXau libXi libXtst make unzip net-tools
```

Validasi:

```bash id="p0v"
getenforce
systemctl status chronyd | grep active
```

---

# PART 1 — USER & GROUP

Membuat user dan group standar Oracle untuk Grid dan Database.

```bash id="p1"
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54324 asmadmin
groupadd -g 54325 asmdba
groupadd -g 54326 asmoper

useradd -u 54321 -g oinstall -G dba,asmdba oracle
useradd -u 54322 -g oinstall -G asmadmin,asmdba,asmoper,dba grid

passwd oracle
passwd grid
```

Validasi:

```bash id="p1v"
id oracle
id grid
```

---

# PART 2 — DIRECTORY STRUCTURE

Menyusun struktur direktori Oracle Base, Grid Home, dan DB Home dengan ownership yang benar.

```bash id="p2"
mkdir -p /u01/app/oracle
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/oracle/product/19.0.0/db

chown -R oracle:oinstall /u01/app/oracle
chown -R grid:oinstall /u01/app/19.0.0
chmod -R 775 /u01/app
```

Validasi:

```bash id="p2v"
ls -ld /u01/app/oracle
ls -ld /u01/app/19.0.0/grid
```

---

# PART 3 — NETWORK

Mengatur resolusi hostname untuk komunikasi RAC (public, private, VIP, SCAN).

```bash id="p3"
192.168.56.11 plvmracdb1
192.168.56.20 plvmracdb2

192.168.56.12 plvmracdb1-vip
192.168.56.21 plvmracdb2-vip

192.168.171.11 plvmracdb1-priv
192.168.171.20 plvmracdb2-priv

192.168.56.90 rac-scan
```

Validasi:

```bash id="p3v"
ping -c 1 plvmracdb2
getent hosts rac-scan
```

---

# PART 4 — DISK ASM

Menyiapkan disk yang akan digunakan oleh ASM.

```bash id="p4"
lsblk
chown grid:asmadmin /dev/sdb /dev/sdc
chmod 660 /dev/sdb /dev/sdc
```

Validasi:

```bash id="p4v"
ls -l /dev/sdb /dev/sdc
```

---

# PART 5 — PASSWORDLESS SSH

Menyiapkan SSH tanpa password antar node dan ke diri sendiri (self SSH).

```bash id="p5reset"
su - grid
rm -rf ~/.ssh
mkdir ~/.ssh
chmod 700 ~/.ssh
```

Node1:

```bash id="p5n1"
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id grid@plvmracdb2
```

Node2:

```bash id="p5n2"
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id grid@plvmracdb1
```

Self SSH (di kedua node):

```bash id="p5self"
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Validasi:

```bash id="p5v"
ssh -o PasswordAuthentication=no plvmracdb1 hostname
ssh -o PasswordAuthentication=no plvmracdb2 hostname
```

---

# PART 6 — /tmp CONFIGURATION

Memastikan direktori /tmp bisa digunakan untuk eksekusi remote oleh Oracle.

```bash id="p6"
chmod 1777 /tmp
mount -o remount,exec /tmp
rm -rf /tmp/*
```

Validasi:

```bash id="p6v"
ssh plvmracdb2 "echo OK > /tmp/test && cat /tmp/test"
```

---

# PART 7 — UNZIP GRID HOME

Menyiapkan binary Grid Infrastructure.

```bash id="p7"
su - grid
unzip LINUX.X64_193000_grid_home.zip -d /u01/app/19.0.0/grid
```

Validasi:

```bash id="p7v"
ls /u01/app/19.0.0/grid/runInstaller
```

---

# PART 8 — ASM AFD

Mengaktifkan ASM Filter Driver dan melabel disk.

```bash id="p8"
export ORACLE_HOME=/u01/app/19.0.0/grid
export PATH=$ORACLE_HOME/bin:$PATH

asmcmd afd_configure
asmcmd afd_label DATA1 /dev/sdb
asmcmd afd_label DATA2 /dev/sdc
```

Validasi:

```bash id="p8v"
asmcmd afd_lsdsk
```

---

# PART 9 — INSTALL GRID

Menjalankan instalasi Grid Infrastructure.

```bash id="p9"
./gridSetup.sh -ignorePrereq -J"-Doracle.install.db.validate.supportedOSCheck=false"
```

Konfigurasi:

* Cluster: RAC
* SCAN: rac-scan
* Network: enp0s8 (public), enp0s9 (private)
* Storage: ASM (AFD)
* Diskgroup: External redundancy

---

# PART 10 — ROOT SCRIPT

Menyelesaikan instalasi dengan hak akses root.

```bash id="p10"
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

---

# PART 11 — UNZIP DB HOME

Menyiapkan binary Oracle Database.

```bash id="p11"
su - oracle
unzip LINUX.X64_193000_db_home.zip -d /u01/app/oracle/product/19.0.0/db
```

Validasi:

```bash id="p11v"
ls /u01/app/oracle/product/19.0.0/db/runInstaller
```

---

# FINAL VALIDATION

Memastikan cluster berjalan normal.

```bash id="p12"
crsctl check cluster -all
olsnodes -n
```

---

# RESULT

RAC 2 node aktif, ASM berjalan dengan AFD, SSH antar node valid, dan cluster siap digunakan.

---

✔ RHEL9 Ready

---
