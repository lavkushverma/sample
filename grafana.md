# 📊 Step‑by‑Step: Configuring Grafana (From Scratch)

---

## 🧭 1. What is Grafana?

Grafana is an open-source analytics and visualization platform.  
It allows you to connect to a data source (like CloudWatch, Prometheus, InfluxDB, or MySQL) and create **beautiful, real‑time dashboards**.

> AWS Context: When working in AWS, Grafana is often used with **CloudWatch**, **Prometheus**, or **Timestream** to visualize metrics from EC2, Lambda, EKS, or any custom app.

---

## 🧩 2. Prerequisites

Before starting, you’ll need:
- 🖥️ A **Linux/Windows/macOS** server (e.g., EC2 instance).  
- 🌐 Internet access (Grafana downloads plugins and contacts data sources).  
- ⚙️ Administrative rights (sudo/root) on that machine.  
- 📦 Data source credentials (like AWS access key for CloudWatch).  

---

## ⚙️ 3. Step 1 — Install Grafana

### **🪟 Option 1: Using a Debian/Ubuntu system**

```bash
# Update packages
sudo apt-get update

# Install dependencies
sudo apt-get install -y apt-transport-https software-properties-common wget

# Add Grafana GPG key
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

# Add Grafana APT repository
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Install Grafana
sudo apt-get update
sudo apt-get install grafana -y
```

### **💻 Option 2: Using Amazon Linux or CentOS**

```bash
sudo yum install -y https://dl.grafana.com/oss/release/grafana-10.4.1-1.x86_64.rpm
sudo yum install grafana
```

### **🧠 Option 3: Docker (quick test environment)**

```bash
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  grafana/grafana-oss
```

---

## 🧨 4. Step 2 — Start Grafana Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Then verify the service:

```bash
sudo systemctl status grafana-server
```

> ✅ Grafana listens on port **3000** by default.

---

## 🌐 5. Step 3 — Access Grafana Web UI

- Open your browser:  
  👉 `http://<your_server_ip>:3000`

- Default credentials:
  - **Username:** `admin`
  - **Password:** `admin`

You’ll be prompted to **set a new password** right away (please do—this saves you a future security headache).

---

## ⚡ 6. Step 4 — Add Your First Data Source

Everything begins with data. Grafana itself stores no data; it visualizes metrics from external systems.

### Example: **Connect AWS CloudWatch**

1. In the Grafana web menu → click ⚙️ **Settings > Data Sources**  
2. Click **Add data source**  
3. Choose **“CloudWatch”**  
4. Enter AWS credentials:
   - **Default region:** like `us-east-1`
   - **Auth provider:** choose `Access & secret key` or `AWS SDK default`
   - **Access Key ID / Secret Access Key**

> 🧩 You can also assign an **IAM Role** to your EC2 instance running Grafana so you don’t need keys.

5. Click **Save & Test** — Grafana will confirm connection success.

### Other popular sources:

| Source | Plugin Name | Docs |
|---------|--------------|------|
| Prometheus | `prometheus` | [Prometheus Plugin Docs](https://grafana.com/docs/grafana/latest/datasources/prometheus/) |
| InfluxDB | `influxdb` | [InfluxDB Plugin Docs](https://grafana.com/docs/grafana/latest/datasources/influxdb/) |
| PostgreSQL/MySQL | `mysql`, `postgres` | [SQL Data Sources](https://grafana.com/docs/grafana/latest/datasources/sql/) |

---

## 📈 7. Step 5 — Create Your First Dashboard

1. Click the **“+” icon > Dashboard**.  
2. Select **“Add new panel.”**
3. On the right, choose your **data source** (CloudWatch, Prometheus, etc.).
4. Write a query:
   - Example (CloudWatch): select metric namespace like `AWS/EC2`
   - Choose metric: `CPUUtilization`
   - Select dimension: `InstanceId`
5. Adjust **visualization type** (Graph, Gauge, Bar, Table, etc.).
6. Click **Apply**.

🎨 Voilà — your first live Grafana panel!

> 💡 Tip: You can clone panels, apply filters, and group them into rows to organize dashboards neatly.

---

## 🧰 8. Step 6 — Customize Dashboard Appearance

Under **Dashboard Settings**:
- Add a **title** (e.g., “EC2 CPU Monitoring”).  
- Set **auto-refresh** (10s, 30s, 1m).  
- Configure **time range** defaults (Last 6 hours, Last 24h, etc.).
- Toggle **variables** (dynamic dropdowns to switch regions/instances easily).

Example of a variable for selecting an EC2 instance:
```
Name: instance
Type: Query
Data Source: CloudWatch
Query: instances()
```

Now your dashboard lets you switch instance metrics dynamically!

---

## 🔒 9. Step 7 — Secure and Manage Access

### a. **Create Users and Teams**
1. Go to ⚙️ → **Users and Access Control**
2. Add users manually or integrate with:
   - **LDAP**
   - **GitHub OAuth**
   - **SAML / Azure AD**
   
### b. **Assign Roles**
- **Viewer:** Read-only access  
- **Editor:** Can modify dashboards  
- **Admin:** Complete control  

### c. **Enable HTTPS (Optional but Important)**

Edit configuration at `/etc/grafana/grafana.ini`:

```ini
[server]
protocol = https
cert_file = /etc/ssl/certs/grafana.crt
cert_key = /etc/ssl/private/grafana.key
```

Restart Grafana:

```bash
sudo systemctl restart grafana-server
```

---

## 📧 10. Step 8 — Configure Alerts (Optional but Powerful)

Grafana can notify you when metrics breach thresholds.

1. Edit any panel → click **Alert** tab.  
2. Add condition (e.g., avg CPU > 80% for 5m).  
3. Configure **alert channel**: email, Slack, PagerDuty, etc.
4. Save and test notification.

Example Slack alert message:
> “🚨 CPUUtilization exceeded 80% for i-097fabc12def34567.”

---

## 🔄 11. Step 9 — Backup and Dashboard Export

Dashboards are stored in `/var/lib/grafana/` by default.  
Use built-in export:
- From dashboard → click **Share → Export → Save JSON**.

You can **import dashboards** using a JSON file or **Grafana.com Dashboard ID** (like “AWS CloudWatch Dashboard ID 7362”).

---

## 🧮 12. Step 10 — Automate Grafana Deployment (IaC Style)

For professional environments, you can automate Grafana setup using:
- **Terraform Grafana provider**  
- **Ansible Grafana role**  
- **Docker Compose**

Example Terraform snippet:
```hcl
resource "grafana_dashboard" "main" {
  config_json = file("${path.module}/dashboard.json")
}
```

---

## 🎯 Quick Recap

| Step | Purpose |
|------|----------|
| 1–2 | Install and start Grafana service |
| 3 | Access via web console (port 3000) |
| 4 | Add data source (e.g., CloudWatch) |
| 5 | Create dashboard and panels |
| 6 | Customize time ranges, queries, and layout |
| 7 | Secure access with users, roles, HTTPS |
| 8 | Add alerts or notifications |
| 9 | Export/backup dashboards |
| 10 | Automate configuration for scale |

---

## 📚 Official Documentation Links

| Topic | Link |
|-------|------|
| Main Docs | [https://grafana.com/docs/grafana/latest/](https://grafana.com/docs/grafana/latest/) |
| Install Grafana | [https://grafana.com/docs/grafana/latest/setup-grafana/installation/](https://grafana.com/docs/grafana/latest/setup-grafana/installation/) |
| Configure Data Sources | [https://grafana.com/docs/grafana/latest/datasources/](https://grafana.com/docs/grafana/latest/datasources/) |
| CloudWatch Integration | [https://grafana.com/docs/grafana/latest/datasources/cloudwatch/](https://grafana.com/docs/grafana/latest/datasources/cloudwatch/) |
| Alerts | [https://grafana.com/docs/grafana/latest/alerting/](https://grafana.com/docs/grafana/latest/alerting/) |
| Authentication | [https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/) |

---

## ☁️ Final Thoughts

For freshers, setting up Grafana is an **essential DevOps muscle** — it connects your metrics to real-world visibility.  

After the first setup, try:
- Connecting multiple data sources.  
- Building team dashboards.  
- Integrating alerts via Slack/Email.  
- Automating setup with Terraform.

> 🌟 Once you see your first live graph dance with EC2 CPU metrics, you’ll feel like a monitoring wizard.
