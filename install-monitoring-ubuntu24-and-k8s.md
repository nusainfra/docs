# Panduan Install Monitoring di VM Ubuntu 24 dan Kubernetes (Prometheus, Grafana, Loki, node-exporter)

**Ringkasan singkat**
- Target: VM Ubuntu 24 (IP: `207.148.122.217`) — install Prometheus (server), Grafana, Loki, node_exporter.
- Di Kubernetes: install **Prometheus (agent mode)** dan **Promtail** menggunakan Helm.
- Dokumen ini berisi langkah per langkah (commands ready-to-run), contoh `systemd` unit, contoh konfigurasi minimal, dan tips keamanan dasar.

> Catatan: jalankan perintah berikut sebagai user dengan hak `sudo` atau `root`.

---

## Daftar isi
1. Prasyarat
2. Persiapan VM (firewall, user, update)
3. Install Prometheus (di VM) — binary & systemd
4. Install node_exporter (di VM)
5. Install Grafana (APT) dan konfigurasi awal
6. Install Loki (di VM) — single-binary mode & systemd
7. Akses dashboard & contoh scraping Prometheus
8. Kubernetes: persiapan Helm
9. Kubernetes: pasang Prometheus (Agent mode) dengan Helm
10. Kubernetes: pasang Promtail dengan Helm
11. Troubleshooting & tips ops

---

## 1) Prasyarat
- VM Ubuntu 24 (IP publik: `207.148.122.217`). Pastikan port yang diperlukan dibuka pada firewall dan provider cloud.
- `sudo`, `curl`, `wget`, `tar`, `git`, `systemd` tersedia.
- Untuk bagian Kubernetes: `kubectl`, `helm` dan akses ke cluster sudah tersedia.

---

## 2) Persiapan VM
```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y curl wget tar gzip gnupg lsb-release apt-transport-https

sudo apt install -y ufw
sudo ufw allow OpenSSH
sudo ufw allow 9090/tcp
sudo ufw allow 3000/tcp
sudo ufw allow 3100/tcp
sudo ufw allow 9100/tcp
sudo ufw enable
```

---

## 3) Install Prometheus (di VM) — binary & systemd

### Download & Setup
```bash
PROM_VER="3.8.0"
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VER}/prometheus-${PROM_VER}.linux-amd64.tar.gz

sudo useradd --no-create-home --shell /bin/false prometheus || true
sudo mkdir -p /etc/prometheus /var/lib/prometheus

sudo tar xvf prometheus-${PROM_VER}.linux-amd64.tar.gz
cd prometheus-${PROM_VER}.linux-amd64
sudo cp prometheus promtool /usr/local/bin/
```

### Config Prometheus `/etc/prometheus/prometheus.yml`
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

### Systemd Service `/etc/systemd/system/prometheus.service`
```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus   --config.file=/etc/prometheus/prometheus.yml   --storage.tsdb.path=/var/lib/prometheus   --web.listen-address=":9090"

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Enable
```bash
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

---

## 4) Install node_exporter (di VM)
```bash
NODE_VER="1.10.2"
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_VER}/node_exporter-${NODE_VER}.linux-amd64.tar.gz
tar xvf node_exporter-${NODE_VER}.linux-amd64.tar.gz
sudo mv node_exporter-${NODE_VER}.linux-amd64/node_exporter /usr/local/bin/

sudo useradd --no-create-home --shell /bin/false node_exporter || true
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### Systemd `/etc/systemd/system/node_exporter.service`
```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Enable
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

---

## 5) Install Grafana (APT)
```bash
sudo wget -q -O - https://packages.grafana.com/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg

echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main"  | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install -y grafana
sudo systemctl enable --now grafana-server
```

Akses Grafana:  
👉 `http://207.148.122.217:3000`  
(User: admin / Password: admin)

---

## 6) Install Loki (Single Binary)

```bash
LOKI_VER="3.6.2"
cd /tmp
wget https://github.com/grafana/loki/releases/download/v${LOKI_VER}/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki

sudo useradd --no-create-home --shell /bin/false loki || true
sudo mkdir -p /etc/loki /var/lib/loki
sudo chown loki:loki /var/lib/loki
```

### Config `/etc/loki/local-config.yaml`
```yaml
common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

limits_config:
  metric_aggregation_enabled: true
  enable_multi_variant_queries: true

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

pattern_ingester:
  enabled: true
  metric_aggregation:
    loki_address: localhost:3100


frontend:
  encoding: protobuf
```

### Systemd `/etc/systemd/system/loki.service`
```ini
[Unit]
Description=Loki Service
Wants=network-online.target
After=network-online.target

[Service]
User=loki
Group=loki
Type=simple
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/local-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Enable
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now loki
sudo systemctl status loki
```

---

## 7) Akses layanan
- Prometheus → `http://207.148.122.217:9090`  
- Grafana → `http://207.148.122.217:3000`  
- Loki API → `http://207.148.122.217:3100`

---

## 8) Kubernetes — Persiapan Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create ns observability
kubectl create ns loki-stack
```

---

## 9) Install Prometheus Agent Mode (Helm Chart)

### File `prometheus-agent-values.yaml`
```yaml
prometheus:
  enabled: true
  agentMode: true
  prometheusSpec:
    remoteWrite:
      - url: "http://207.148.122.217:9090/api/v1/write"
grafana:
  enabled: false
kubeStateMetrics:
  enabled: true
nodeExporter:
  enabled: true
  operatingSystems:
    linux:
      enabled: true
    aix:
      enabled: false
    darwin:
      enabled: false
defaultRules:
  create: false
alertmanager:
  enabled: false
```

### Install
```bash
helm upgrade --install prometheus-agent prometheus-community/kube-prometheus-stack   -n observability   -f prometheus-agent-values.yaml
```

---

## 10) Install Promtail (Helm)

### File `promtail-values.yaml`
```yaml
config:
  clients:
    - url: http://207.148.122.217:3100/loki/api/v1/push
```

### Install
```bash
helm install promtail grafana/promtail -n loki-stack -f promtail-values.yaml
```

---

## 11) Troubleshooting

- Prometheus agent error: pastikan `ruleFiles: []`.
- Cek log:
```bash
sudo journalctl -u prometheus -f
kubectl logs -n observability <pod>
```
- Pastikan waktu sinkron (NTP).
- Amankan Grafana dengan password baru + HTTPS.

---

Selesai. Semoga membantu!
