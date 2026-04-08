# Oracle Monitoring Setup (Prometheus + Grafana + Docker Exporter)

---

# 📌 OVERVIEW

Dokumentasi ini menjelaskan setup monitoring Oracle yang scalable untuk:

* Oracle RAC
* Oracle Non-RAC
* Future database servers

Menggunakan:

* Prometheus (collector)
* Grafana (visualization)
* Node Exporter (OS metrics)
* Oracle Exporter (database metrics via Docker)

---

# 🧱 ARSITEKTUR

```
grafanavm (Monitoring Server)
 ├── Prometheus
 ├── Grafana

DB Servers
 ├── Node Exporter
 ├── Oracle Exporter (Docker)
```

---

# ENVIRONMENT 
| Server     | IP            |
| ---------- | ------------- |
| grafanavm  | 192.168.56.44 |
| plvmracdb1 | 192.168.56.11 |
| plvmracdb2 | 192.168.56.20 |
| plvmracdb3 | 192.168.56.30 |

---

#  PART 1 — INSTALL NODE EXPORTER (SEMUA SERVER)

## 📍 Lokasi: `/opt/node_exporter`

---

## STEP 1 — Download

```bash
cd /opt
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xvf node_exporter-*.tar.gz
mv node_exporter-* node_exporter
```

Penjelasan:

* Download binary node_exporter
* Extract dan rename folder agar konsisten

---

## STEP 2 — Create Service

```bash
vim /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
ExecStart=/opt/node_exporter/node_exporter

[Install]
WantedBy=multi-user.target
```

Penjelasan:

* Menjalankan node_exporter sebagai service systemd

---

## STEP 3 — Start Service

```bash
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
```

---

## STEP 4 — Validasi

```bash
curl http://localhost:9100/metrics
```

---

# 🚀 PART 2 — INSTALL DOCKER (SERVER DATABASE SAJA)

---

## STEP 1 — Enable Repo

```bash
yum install -y yum-utils
yum-config-manager --enable ol7_addons
```

---

## STEP 2 — Install Docker

```bash
yum install -y docker
```

---

## STEP 3 — Start Docker

```bash
systemctl start docker
systemctl enable docker
```

---

## STEP 4 — Validasi

```bash
docker run hello-world
```

---

# 🚀 PART 3 — ORACLE EXPORTER (DOCKER)

---

## 📍 Tidak butuh folder khusus (container-based)

---

## STEP 1 — Pull Image

```bash
docker pull ghcr.io/iamseth/oracledb_exporter:0.5.0
```

---

## STEP 2 — RUN EXPORTER (RAC)

Jalankan di SETIAP NODE RAC

```bash
docker run -d \
  --name oracle-exporter \
  -p 9161:9161 \
  -e DATA_SOURCE_NAME="monitor/monitor@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=rac-scan)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=byteworks)))" \
  ghcr.io/iamseth/oracledb_exporter:0.5.0
```

Penjelasan:

* Menggunakan SCAN untuk high availability
* 1 container per node

---

## STEP 3 — RUN EXPORTER (NON-RAC)

```bash
docker run -d \
  --name oracle-exporter \
  -p 9161:9161 \
  -e DATA_SOURCE_NAME="monitor/monitor@192.168.56.50:1521/orcl" \
  ghcr.io/iamseth/oracledb_exporter:0.5.0
```

Penjelasan:

* Langsung connect ke IP server

---

## STEP 4 — VALIDASI EXPORTER

```bash
curl http://localhost:9161/metrics
```

---

## STEP 5 — AUTO START CONTAINER

```bash
docker update --restart=always oracle-exporter
```

Penjelasan:

* Container otomatis hidup saat reboot

---

# 🚀 PART 4 — PROMETHEUS CONFIG

---

## 📍 Lokasi file

```bash
/opt/prometheus/etc/prometheus.yml
```

---

## STEP 1 — Edit Config

```bash
vim /opt/prometheus/etc/prometheus.yml
```

---

## STEP 2 — Tambahkan Target

```yaml
global:
  scrape_interval: 15s

scrape_configs:

  - job_name: 'node-exporter'
    static_configs:
      - targets:
        - '192.168.56.11:9100'
        - '192.168.56.20:9100'
        - '192.168.56.30:9100'
        - '192.168.56.44:9100'

  - job_name: 'oracle'
    static_configs:
      - targets:
        - '192.168.56.11:9161'
        - '192.168.56.20:9161'
        - '192.168.56.30:9161'
```

---

## STEP 3 — Restart Prometheus

```bash
systemctl restart prometheus
```

---

## STEP 4 — Validasi

```
http://192.168.56.44:9090/targets
```

---

# 🚀 PART 5 — TAMBAH DATABASE BARU

---

# 🟢 CASE 1 — NON-RAC

## STEP 1 — Install Docker

(sama seperti sebelumnya)

---

## STEP 2 — Run Exporter

```bash
docker run -d \
  --name oracle-exporter \
  -p 9161:9161 \
  -e DATA_SOURCE_NAME="monitor/monitor@192.168.56.60:1521/newdb" \
  ghcr.io/iamseth/oracledb_exporter:0.5.0
```

---

## STEP 3 — Tambah ke Prometheus

```yaml
- '192.168.56.60:9161'
```

---

# 🔴 CASE 2 — RAC (DETAIL)

---

## STEP 1 — Jalankan di SETIAP NODE

Node 1:

```bash
docker run -d -p 9161:9161 ...
```

Node 2:

```bash
docker run -d -p 9161:9161 ...
```

Node 3:

```bash
docker run -d -p 9161:9161 ...
```

---

## STEP 2 — Gunakan SCAN

```text
HOST=rac-scan
```

---

## STEP 3 — Tambahkan ke Prometheus

```yaml
- 'node1:9161'
- 'node2:9161'
- 'node3:9161'
```

---

# 🚀 TROUBLESHOOTING

---

## Cek container

```bash
docker ps
```

---

## Cek log

```bash
docker logs oracle-exporter
```

---

## Test DB

```bash
sqlplus monitor/monitor@rac-scan
```

---

## Cek port

```bash
netstat -tulnp | grep 9161
```

---

# BEST PRACTICE

* 1 exporter per DB server
* RAC tetap per node
* Gunakan SCAN untuk RAC
* Gunakan IP untuk non-RAC
* Prometheus hanya 1 server

---

# FINAL RESULT

✔ Monitoring semua server
✔ Support RAC & non-RAC
✔ Mudah scaling
✔ Auto start
✔ Production ready

---
