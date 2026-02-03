---

# 🧭 Step‑by‑Step: Configure Prometheus from Scratch

---

## 🌩️ 1. What Is Prometheus?

Prometheus is an **open‑source monitoring and alerting toolkit** originally built by SoundCloud.  

It collects numerical data (metrics) from your systems and applications, stores them in a **time‑series database**, and lets you query it using **PromQL (Prometheus Query Language)**.

It’s lightweight, reliable, and integrates beautifully with **Grafana**, **Kubernetes**, and **AWS CloudWatch exporters**.

---

## 🧩 2. Prerequisites

You’ll need:
- A **Linux** instance (e.g., Ubuntu, Debian, Amazon Linux, or CentOS).  
- `curl` and `wget` installed.  
- Root or `sudo` access.  
- Optionally: your **application or system metrics endpoints** (`/metrics`) ready to test.  

For training and practice, a small **t2.micro EC2** (or local VM) is perfect.

---

## ⚙️ 3. Step 1 — Install Prometheus

Let’s get it running manually first to understand the structure.

### **🪄 Download and Extract**

```bash
# Update system and install required packages
sudo apt-get update -y
sudo apt-get install wget tar -y

# Download the latest Prometheus package (check version at prometheus.io)
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz

# Extract the archive
tar -xvf prometheus-2.52.0.linux-amd64.tar.gz

# Move it into /usr/local/bin for global access
sudo mv prometheus-2.52.0.linux-amd64 /etc/prometheus
sudo ln -s /etc/prometheus/prometheus /usr/local/bin/prometheus
sudo ln -s /etc/prometheus/promtool /usr/local/bin/promtool
```

---

## 🧰 4. Step 2 — Directory Setup

Organize directories cleanly:

```bash
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown -R ubuntu:ubuntu /etc/prometheus /var/lib/prometheus
```

Then verify folder structure:
```
/etc/prometheus/
 ├── prometheus.yml   ← main config file
 ├── consoles/
 ├── console_libraries/
```

---

## ⚡ 5. Step 3 — Configure prometheus.yml

This configuration tells Prometheus **what to scrape** and **how often**.

Open the file:
```bash
sudo nano /etc/prometheus/prometheus.yml
```

Paste this starter configuration:

```yaml
# my global configuration
global:
  scrape_interval: 15s  # How often to scrape targets (default = 1m)

# Enable metrics about Prometheus itself
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Example: Linux node metrics using node_exporter
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  # Example: your application metrics endpoint
  - job_name: 'myapp'
    static_configs:
      - targets: ['localhost:8080']
```

> 💡 Prometheus **pulls** metrics from targets (it doesn’t wait for metrics to be pushed). Each `target` must expose a `/metrics` HTTP endpoint.

Save and exit (`Ctrl + O`, then `Ctrl + X`).

---

## 🧠 6. Step 4 — Create Prometheus User and Service

You’ll want Prometheus to run as a background service.

```bash
sudo useradd --no-create-home --shell /bin/false prometheus

sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

Now create a systemd service file:

```bash
sudo tee /etc/systemd/system/prometheus.service > /dev/null << 'EOF'
[Unit]
Description=Prometheus Monitoring Service
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Then enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

---

## 🌐 7. Step 5 — Access Prometheus Web UI

Open your browser:
```
http://<your-server-ip>:9090
```

You’ll see the Prometheus dashboard!

Try a quick test query:
```
up
```
This shows which targets are currently being scraped (value `1` means “reachable”).

---

## 🧭 8. Step 6 — Add a Node Exporter (for System Metrics)

Prometheus itself collects data from **exporters** (agents) that expose system metrics.

### **Install Node Exporter**
```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
```

Create a systemd service for node_exporter:

```bash
sudo tee /etc/systemd/system/node_exporter.service > /dev/null << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Start service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

Node Exporter will now expose system metrics at:
```
http://localhost:9100/metrics
```

And Prometheus (with the config above) will automatically start scraping it.

---

## 📈 9. Step 7 — Verify Data Collection

In Prometheus web console:
- Go to “**Status → Targets**”
- You should see:
  - `prometheus` target (port 9090)
  - `node_exporter` target (port 9100)
- Status = **UP**

Then in the **Graph** tab, search:
```
node_cpu_seconds_total
```

or

```
node_memory_MemAvailable_bytes
```

You’ll see raw data points flowing in. That’s success!

---

## 🎨 10. Step 8 — Integrate with Grafana

Now that Prometheus is scraping data, let’s visualize it.

Inside Grafana UI:
1. Go to ⚙️ → *Data Sources* → *Add Data Source*  
2. Select **Prometheus**.  
3. URL: `http://<your_prometheus_server_ip>:9090`
4. Click **Save & Test**.  

From here, you can import **pre‑built Grafana dashboards** for Prometheus and Node Exporter:
- Official ID for Linux Node Exporter dashboard: **1860**
- (In Grafana → “+ Import” → enter 1860 → choose Prometheus data source → load.)

---

## 🧠 11. Step 9 — Set Up Alerts (Optional)

Prometheus includes a separate service called **Alertmanager**.

You configure alerts in Prometheus like this:

Append to `prometheus.yml`:

```yaml
rule_files:
  - "alerts.yml"
```

Create `/etc/prometheus/alerts.yml`:
```yaml
groups:
  - name: instance_alerts
    rules:
      - alert: HighCPUUsage
        expr: node_cpu_seconds_total{mode="system"} > 0.80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Instance {{ $labels.instance }} has high CPU usage"
          description: "CPU usage is above 80% for more than 2 minutes."
```

Restart Prometheus:
```bash
sudo systemctl restart prometheus
```

Next, configure Alertmanager to send emails or Slack alerts (later step).

---

## 🛡️ 12. Step 10 — Secure Prometheus (Optional)

Out of the box, Prometheus has no authentication. For production:
- Run behind Nginx reverse proxy with **basic auth**.  
- Restrict network access via **AWS Security Groups** or firewalls.  
- Enable HTTPS on proxy level.  

Simple Nginx reverse proxy example:

```bash
sudo apt install nginx apache2-utils -y
sudo htpasswd -c /etc/nginx/.htpasswd prom_user
```

Edit `/etc/nginx/sites-enabled/prometheus.conf`:

```nginx
server {
  listen 9443;
  server_name your_domain.com;

  auth_basic "Prometheus Access";
  auth_basic_user_file /etc/nginx/.htpasswd;

  location / {
    proxy_pass http://localhost:9090;
  }
}
```

Restart Nginx. Now Prometheus UI will require login!

---

## 🧮 13. Step 11 — Backup and Maintenance

Prometheus stores data in:
```
/var/lib/prometheus/
```

Back up regularly with `rsync` or snapshots.

For performance:
- Avoid too small scrape intervals (<10s).  
- Use **recording rules** to pre‑aggregate metrics.  
- Clean old metrics with retention flag:
  ```bash
  --storage.tsdb.retention.time=15d
  ```

---

## 📘 Example Summary (Mini‑Architecture)

| Layer | Component | Port | Purpose |
|--------|------------|------|----------|
| **Core** | Prometheus Server | 9090 | Collects + stores time‑series metrics |
| **Agent** | Node Exporter | 9100 | Exposes OS metrics |
| **Visualization** | Grafana | 3000 | Visual analytics layer |
| **Alerts (optional)** | Alertmanager | 9093 | Sends notifications |

---

## 📚 Official Documentation Links

| Area | Link |
|------|------|
| Prometheus Overview | [https://prometheus.io/docs/introduction/overview/](https://prometheus.io/docs/introduction/overview/) |
| Installation Guide | [https://prometheus.io/docs/prometheus/latest/installation/](https://prometheus.io/docs/prometheus/latest/installation/) |
| Configuration File | [https://prometheus.io/docs/prometheus/latest/configuration/configuration/](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) |
| Query Language (PromQL) | [https://prometheus.io/docs/prometheus/latest/querying/basics/](https://prometheus.io/docs/prometheus/latest/querying/basics/) |
| Node Exporter | [https://prometheus.io/docs/guides/node-exporter/](https://prometheus.io/docs/guides/node-exporter/) |
| Alertmanager | [https://prometheus.io/docs/alerting/latest/overview/](https://prometheus.io/docs/alerting/latest/overview/) |

---
