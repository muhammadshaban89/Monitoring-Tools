**clean, complete, production‑ready guide** for setting up **Prometheus alerting + Alertmanager + Email notifications**, with each step explained clearly.



#  **Prometheus Email Alerting Setup (Prometheus + Alertmanager + SMTP)**  
This is the standard, recommended architecture:

```
Prometheus  →  Alertmanager  →  Email (SMTP)
```

Prometheus **detects alert conditions**, but **Alertmanager sends the actual email**.  
This separation is intentional and part of Prometheus design.

---

# 1️ Install Alertmanager (on RHEL 9.5)

Alertmanager is a separate binary from Prometheus.

```bash
cd /tmp

# Get latest Alertmanager version
LATEST=$(curl -s https://api.github.com/repos/prometheus/alertmanager/releases/latest | grep tag_name | cut -d '"' -f 4)

# Download correct tarball
curl -LO https://github.com/prometheus/alertmanager/releases/download/${LATEST}/alertmanager-${LATEST#v}.linux-amd64.tar.gz

# Extract
tar -xvf alertmanager-${LATEST#v}.linux-amd64.tar.gz

# Move binaries (overwrite if already exist)
sudo mv -f alertmanager-${LATEST#v}.linux-amd64/alertmanager /usr/local/bin/
sudo mv -f alertmanager-${LATEST#v}.linux-amd64/amtool /usr/local/bin/

# Create config directory
sudo mkdir -p /etc/alertmanager

# Move default config (overwrite if exists)
sudo mv -f alertmanager-${LATEST#v}.linux-amd64/alertmanager.yml /etc/alertmanager/

# Create data directory (REQUIRED)
sudo mkdir -p /etc/alertmanager/data

# Create alertmanager user if not exists
id alertmanager &>/dev/null || sudo useradd --no-create-home --shell /sbin/nologin alertmanager

# Set permissions
sudo chown -R alertmanager:alertmanager /etc/alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/amtool
```

**Why:**  
Alertmanager handles deduplication, grouping, routing, silencing, and sending notifications (email, Slack, PagerDuty, etc.) .

---

# 2️ Create Alertmanager systemd service

```bash
sudo tee /etc/systemd/system/alertmanager.service > /dev/null << 'EOF'
[Unit]
Description=Alertmanager Service
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager
```

Alertmanager UI:  
👉 `http://<RHEL-IP>:9093`

---

# 3️ Configure Alertmanager for Email (SMTP)

Edit:

```bash
sudo nano /etc/alertmanager/alertmanager.yml
```

Use this template:

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'your-email@gmail.com'
  smtp_auth_username: 'your-email@gmail.com'
  smtp_auth_password: 'your-app-password'

route:
  receiver: 'email-alerts'

receivers:
  - name: 'email-alerts'
    email_configs:
      - to: 'destination-email@example.com'
        send_resolved: true
```

**Explanation:**  
- `smtp_smarthost` → SMTP server + port  
- `smtp_from` → sender email  
- `smtp_auth_username/password` → login credentials  
- `route` → default receiver  
- `email_configs` → where Alertmanager sends alerts  

Alertmanager supports routing trees, grouping, and inhibition rules via this config file .

Restart:

```bash
sudo systemctl restart alertmanager
```

---

# 4️ Tell Prometheus to use Alertmanager

Edit:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]
```

Restart Prometheus:

```bash
sudo systemctl restart prometheus
```

---

# 5️ Create Alert Rules (CPU, RAM, Node Down, etc.)

Create rule file:

```bash
sudo mkdir -p /etc/prometheus/rules
sudo nano /etc/prometheus/rules/node_alerts.yml
```

Example rules:

```yaml
groups:
- name: node_alerts
  rules:

  - alert: NodeDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Node is down"
      description: "The node {{ $labels.instance }} is unreachable."

  - alert: HighCPU
    expr: avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) * 100 > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage"
      description: "CPU usage > 80% on {{ $labels.instance }}"

  - alert: HighMemoryUsage
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage"
      description: "Memory usage > 80% on {{ $labels.instance }} (current value: {{ $value }}%)"
```

Now include rule file in Prometheus config:

```yaml
rule_files:
  - "/etc/prometheus/rules/*.yml"
```

Restart:

```bash
sudo systemctl restart prometheus
```

---

# 6️ Test Alerts

Prometheus UI → **Alerts**  
Alertmanager UI → **Alerts**  
You should see firing alerts when conditions match.

- For example putt dummy load on CPU by:
```
for i in {1..4}; do while :; do :; done & done
```

- You will see high CPU usage on DashBoard.
- To remove dummy load:
```
killall bash
```

- You can put dummy load on memory:
```
stress --vm 2 --vm-bytes 1G --timeout 60s
```
---

# 7️ Firewall + SELinux for Alertmanager

### Firewall

```bash
sudo firewall-cmd --permanent --add-port=9093/tcp
sudo firewall-cmd --reload
```

### SELinux

```bash
sudo semanage port -a -t http_port_t -p tcp 9093
```

---

# 8️ Gmail SMTP Notes (Important)

Google requires an **App Password** if 2FA is enabled.  
Use that in `smtp_auth_password`.

---

#  How Alerts Flow (Simple Diagram)

```
[Prometheus]
   |
   |  (sends alert when rule fires)
   v
[Alertmanager]
   |
   |  (routes, groups, deduplicates)
   v
[Email / Slack / PagerDuty / etc.]
```

This matches the official alerting workflow described in the sources you triggered .

---


