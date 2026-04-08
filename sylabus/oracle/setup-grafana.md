# Prometheus + Grafana + Node Exporter Setup (Oracle Linux 7.9)

## Environment

| Parameter          | Value                                |
| ------------------ | ------------------------------------ |
| Monitoring Server  | grafanavm (192.168.56.44)            |
| RAC Nodes          | plvmracdb1 / plvmracdb2 / plvmracdb3 |
| OS                 | Oracle Linux 7.9                     |
| Prometheus Port    | 9090                                 |
| Grafana Port       | 3000                                 |
| Node Exporter Port | 9100                                 |

---

# ARCHITECTURE

```
Node Exporter (All Servers)
        ↓
Prometheus (grafanavm)
        ↓
Grafana (grafanavm)
```

---

# PART 1 — PREPARATION (grafanavm)

```bash
yum clean all
yum makecache
yum update -y
yum install -y wget curl vim net-tools
```

---

# PART 2 — INSTALL PROMETHEUS

## 1. Create User

```bash
useradd --no-create-home --shell /bin/false prometheus
```

---

## 2. Download & Extract

```bash
cd /opt
wget https://github.com/prometheus/prometheus/releases/download/v2.51.2/prometheus-2.51.2.linux-amd64.tar.gz
tar -xvf prometheus-*.tar.gz
mv prometheus-2.51.2.linux-amd64 prometheus
```

---

## 3. Permission

```bash
chown -R prometheus:prometheus /opt/prometheus
```

---

## 4. Configuration

```bash
mkdir -p /opt/prometheus/etc
vim /opt/prometheus/etc/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s

scrape_configs:

  - job_name: 'grafana-server'
    static_configs:
      - targets: ['192.168.56.44:9100']

  - job_name: 'rac-nodes'
    static_configs:
      - targets:
        - '192.168.56.11:9100'
        - '192.168.56.20:9100'
        - '192.168.56.30:9100'
```

---

## 5. Service

```bash
vim /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/etc/prometheus.yml

[Install]
WantedBy=multi-user.target
```

---

## 6. Start

```bash
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
```

---

## 7. Validation

```
http://192.168.56.44:9090
```

---

# PART 3 — INSTALL NODE EXPORTER (ALL SERVERS)

## 1. Download

```bash
cd /opt
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xvf node_exporter-*.tar.gz
mv node_exporter-1.8.1.linux-amd64 node_exporter
```

---

## 2. Service

```bash
vim /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=root
ExecStart=/opt/node_exporter/node_exporter

[Install]
WantedBy=default.target
```

---

## 3. Start

```bash
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
```

---

## 4. Validation

```
http://IP_SERVER:9100/metrics
```

---

# PART 4 — INSTALL GRAFANA (grafanavm)

```bash
yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-10.4.2-1.x86_64.rpm
```

---

## Start Service

```bash
systemctl daemon-reload
systemctl enable grafana-server
systemctl start grafana-server
```

---

## Access

```
http://192.168.56.44:3000
```

Login:

```
admin / admin
```

---

# PART 5 — CONNECT PROMETHEUS TO GRAFANA

1. Settings → Data Sources
2. Add → Prometheus
3. URL:

```
http://localhost:9090
```

4. Save & Test

---

# PART 6 — IMPORT DASHBOARD

Dashboard ID:

```
1860
```

Steps:

* Dashboard → Import
* Input ID
* Select Prometheus

---

# PART 7 — VALIDATION

Prometheus Targets:

```
http://192.168.56.44:9090/targets
```

Expected:

* grafana-server → UP
* rac-nodes → UP

---

# PART 8 — ADD NEW SERVER (SCALABLE)

## 1. Install Node Exporter di server baru

(same as PART 3)

---

## 2. Edit Prometheus Config

```bash
vim /opt/prometheus/etc/prometheus.yml
```

Tambahkan:

```yaml
- '192.168.56.xx:9100'
```

---

## 3. Restart Prometheus

```bash
systemctl restart prometheus
```

---

# PART 9 — TROUBLESHOOTING

## Target DOWN

### Check Node Exporter

```bash
systemctl status node_exporter
```

---

### Check Port

```bash
ss -tulnp | grep 9100
```

---

### Test Connectivity

```bash
curl http://IP:9100/metrics
```

---

### Restart Prometheus

```bash
systemctl restart prometheus
```

---

# SUMMARY

* Prometheus digunakan untuk scraping metrics
* Node Exporter diinstall di semua server
* Grafana digunakan sebagai dashboard visualization
* Sistem scalable, cukup tambah server dan update config
* Monitoring mencakup CPU, Memory, Disk, Network
