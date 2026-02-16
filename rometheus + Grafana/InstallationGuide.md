**How to Deploy**



## 1. Architecture

- **RHEL 9.5**  
  Runs **Prometheus + Grafana + node_exporter**  
  → This is  *monitoring server*.

- **CentOS**  
  Runs **node_exporter**  
  → Exposes Linux metrics to Prometheus.

- **Windows Server 2022**  
  Runs **windows_exporter**  
  → Exposes Windows metrics to Prometheus.

Prometheus pulls metrics from exporters; Grafana reads from Prometheus and shows dashboards.

---

## 2. On RHEL 9.5 – base preparation

```bash
sudo dnf update -y
sudo dnf install -y wget tar
```

**Explanation:**

- **`dnf update -y`**  
  Updates all installed packages to latest versions—security fixes, bug fixes, newer libraries.
- **`dnf install wget tar`**  
  `wget` is used to download files from the internet.  
  `tar` is used to extract `.tar.gz` archives (Prometheus and node_exporter come this way).

---

## 3. Install Prometheus on RHEL 9.5

### 3.1 Create Prometheus user and directories

```bash
sudo useradd --no-create-home --shell /sbin/nologin prometheus
sudo mkdir /etc/prometheus /var/lib/prometheus
```

**Explanation:**

- **Dedicated user (`prometheus`)**  
  - `--no-create-home`: no home directory, because it’s a service account.  
  - `--shell /sbin/nologin`: user cannot log in interactively → security.
- **`/etc/prometheus`**  
  - Standard place for configuration files.
- **`/var/lib/prometheus`**  
  - Storage location for Prometheus time‑series data (metrics).

---

### 3.2 Download and install Prometheus

```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/latest/download/prometheus-*.linux-amd64.tar.gz
tar -xvf prometheus-*.linux-amd64.tar.gz
cd prometheus-*.linux-amd64

sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles console_libraries /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

**Explanation:**

- **`cd /tmp`**  
  Use a temporary working directory for downloads.
- **`curl -LO ...`**  
  Downloads the latest Prometheus Linux binary tarball.
- **`tar -xvf`**  
  Extracts the archive into a folder like `prometheus-2.x.x.linux-amd64`.
- **`mv prometheus promtool /usr/local/bin/`**  
  Places binaries in a standard PATH directory so you can run `prometheus` from anywhere.
- **`mv consoles console_libraries /etc/prometheus/`**  
  These are built‑in web console templates used by Prometheus UI.
- **`mv prometheus.yml /etc/prometheus/prometheus.yml`**  
  Moves the default config file to the config directory.
- **`chown -R prometheus:prometheus ...`**  
  Gives ownership of config and data directories to the `prometheus` user so the service can read/write safely without root.

---

### 3.3 Create Prometheus systemd service

```bash
sudo tee /etc/systemd/system/prometheus.service > /dev/null << 'EOF'
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF
```
- SELinux to permissive permanently:
```
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config && setenforce 0
```
- if You want to set Selinux to Enforcing , then:
```
sudo semanage port -a -t http_port_t -p tcp 9090   # Prometheus
sudo semanage port -a -t http_port_t -p tcp 3000   # Grafana
sudo semanage port -a -t http_port_t -p tcp 9100   # node_exporter
```
- Reload Deamon and enable services.

```
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

**Explanation:**

- **`/etc/systemd/system/prometheus.service`**  
  Defines how systemd should start/stop Prometheus.
- **`Wants=network-online.target` / `After=network-online.target`**  
  Ensures Prometheus starts after the network is up (it needs to scrape remote targets).
- **`User=prometheus` / `Group=prometheus`**  
  Runs the process as the non‑root service user you created.
- **`ExecStart=...`**  
  - `--config.file`: tells Prometheus where to find its config.  
  - `--storage.tsdb.path`: where to store metrics data.  
  - `--web.console.*`: where to find console templates.
- **`daemon-reload`**  
  Tells systemd to re‑scan unit files after creating a new one.
- **`enable --now`**  
  Enables auto‑start at boot and starts it immediately.
- **`status`**  
  Confirms it’s running and shows logs if something failed.

Prometheus UI: `http://<RHEL-IP>:9090`

---

## 4. Install node_exporter on RHEL 9.5

This lets you monitor the RHEL monitoring server itself.

### 4.1 User + binary

```bash
sudo useradd --no-create-home --shell /sbin/nologin node_exporter

cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-*.linux-amd64.tar.gz
tar -xvf node_exporter-*.linux-amd64.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

**Explanation:**

- **`node_exporter` user**  
  Same security pattern: non‑login service account.
- **Download + extract**  
  Gets the latest node_exporter binary.
- **Move to `/usr/local/bin`**  
  Makes it globally executable.
- **`chown`**  
  Ensures the `node_exporter` user owns the binary.

---

### 4.2 systemd service

```bash
sudo tee /etc/systemd/system/node_exporter.service > /dev/null << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```
- Reload Daemon and start services:
```
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

- Add Firewall Rules:
```
sudo firewall-cmd --permanent --add-port=9090/tcp   # Prometheus
sudo firewall-cmd --permanent --add-port=3000/tcp   # Grafana
sudo firewall-cmd --permanent --add-port=9100/tcp   # node_exporter
sudo firewall-cmd --reload
```

**Explanation:**

- **Service unit**  
  Defines node_exporter as a systemd service.
- **`After=network.target`**  
  Needs network stack up, but not necessarily full “online” state.
- **`ExecStart=/usr/local/bin/node_exporter`**  
  Starts exporter on default port `9100`.
- **`enable --now`**  
  Auto‑start at boot + start now.
- Add firewall rules to allow ports. 9090/tcp 3000/tcp & 9100/tcp.
-  `firewall-cmd --permanent` adds rules to the persistent configuration.
- `--reload` applies them without reboot.
- You may restrict sources (e.g., only your admin IP) later for security.
  
- **Metrics URL**  
  `http://<RHEL-IP>:9100/metrics` → raw metrics Prometheus will scrape.

---

## 5. Install Grafana on RHEL 9.5

### 5.1 Add Grafana repo and install

```bash
sudo tee /etc/yum.repos.d/grafana.repo > /dev/null << 'EOF'
[grafana]
name=Grafana OSS
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
EOF

sudo dnf install -y grafana
```

**Explanation:**

- **Custom repo file**  
  - Tells `dnf` where to fetch Grafana packages from (official Grafana OSS repo).
  - `gpgcheck` and `gpgkey` ensure packages are signed and trusted.
- **`dnf install grafana`**  
  Installs Grafana server and dependencies using the package manager (cleaner than manual tarball).

---

### 5.2 Start Grafana

```bash
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

**Explanation:**

- **`grafana-server` service**  
  Runs the Grafana web UI on port `3000`.
- **`enable --now`**  
  Auto‑start at boot and start immediately.
- **UI**  
  Access via `http://<RHEL-IP>:3000` → login `admin / admin` (then change password).

---

### Client Side
---------------

## 6. Install node_exporter on CentOS host(s)

Repeat on each CentOS server you want to monitor.

### 6.1 Prepare

```bash
sudo yum install -y wget tar   # or dnf on newer CentOS Stream
sudo useradd --no-create-home --shell /sbin/nologin node_exporter
```

**Explanation:**

- Installs tools needed to download/extract.
- Creates the same `node_exporter` service user as on RHEL for consistency and security.

---

### 6.2 Download + install

```bash
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-*.linux-amd64.tar.gz
tar -xvf node_exporter-*.linux-amd64.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

**Explanation:**

- Same pattern as RHEL: download, extract, move binary, set ownership.

---

### 6.3 systemd service

```bash
sudo tee /etc/systemd/system/node_exporter.service > /dev/null << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```
- Reload Daem and Start Services:
```
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

**Explanation:**

- Identical service definition → easier to manage across Linux nodes.
- Exposes metrics on `http://<CENTOS-IP>:9100/metrics`.

You must ensure the CentOS firewall allows port `9100` from the RHEL Prometheus server.

- Check Selinux and Set to permissive or Set boolean in case you want to Enforce it.

```
sestatus

#Set it permessive :

sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config && setenforce 0

# I you wand Enforcing ,then allow & Verify

sudo semanage port -a -t http_port_t -p tcp 9100
sudo semanage port -l | grep 9100

```

- Add Firewall Rules:
```
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
```
- Restart the servivice:
```
sudo systemctl restart node_exporter
```
---

## 7. Install windows_exporter on Windows Server 2022

### 7.1 Download MSI

- On Windows Server 2022, open a browser and download the latest `windows_exporter-<version>-amd64.msi` from GitHub.

**Explanation:**

- `windows_exporter` is the Windows equivalent of node_exporter—exports CPU, memory, disk, network, etc., as Prometheus metrics.

---

### 7.2 Install as service

- Run the MSI:
  - Click through the wizard.
  - Keep default port (usually `9182`) unless you have a reason to change.
  - Finish installation.

**Explanation:**

- The MSI automatically:
  - Installs the binary.
  - Registers a Windows service named `windows_exporter`.
  - Configures it to start at boot.

---

### 7.3 Verify and open firewall

Check metrics:

```text
http://<WIN-SERVER-IP>:9182/metrics
```

If blocked, open firewall:

```powershell
New-NetFirewallRule -DisplayName "windows_exporter" -Direction Inbound -Protocol TCP -LocalPort 9182 -Action Allow
```

**Explanation:**

- The URL shows raw metrics in Prometheus format.
- Firewall rule allows Prometheus (on RHEL) to reach the exporter on port `9182`.

---

## 8. Configure Prometheus to scrape all targets

Edit config on RHEL:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add/adjust:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "rhel-node"
    static_configs:
      - targets: ["<RHEL-IP>:9100"]

  - job_name: "centos-nodes"
    static_configs:
      - targets:
          - "<CENTOS1-IP>:9100"
          - "<CENTOS2-IP>:9100"

  - job_name: "windows-servers"
    static_configs:
      - targets:
          - "<WIN2022-IP>:9182"
```

Then:

```bash
sudo systemctl restart prometheus
```

**Explanation:**

- **`global.scrape_interval: 15s`**  
  Prometheus will scrape all targets every 15 seconds.
- **`job_name`**  
  Logical grouping of targets (RHEL, CentOS, Windows).
- **`targets`**  
  List of `<IP>:<port>` where exporters are listening.
- **`restart prometheus`**  
  Reloads the config so new targets are picked up.
- In Prometheus UI → **Status → Targets**: you should see all jobs and targets as **UP**.

---

## 9. Connect Grafana to Prometheus

In Grafana (`http://<RHEL-IP>:3000`):

1. Go to **Configuration → Data sources → Add data source**.
2. Select **Prometheus**.
3. Set URL:

   ```text
   http://localhost:9090
   ```

4. Click **Save & test**.

**Explanation:**

- Grafana needs a data source to query metrics.
- Prometheus is that data source.
- Using `localhost` is fine because Grafana and Prometheus are on the same RHEL server.

---

## 10. Import dashboards

In Grafana:

- Go to **Dashboards → Import**.
- For Linux nodes: use dashboard ID **1860** (Node Exporter Full).
- For Windows: search “windows_exporter” dashboards and import by ID.

**Explanation:**

- These dashboards are pre‑built visualizations for node_exporter/windows_exporter metrics.
- Saves you from manually building panels and queries.

---


## 11. Firewall checklist (RHEL side)

```bash
sudo firewall-cmd --permanent --add-port=9090/tcp   # Prometheus UI
sudo firewall-cmd --permanent --add-port=3000/tcp   # Grafana UI
sudo firewall-cmd --permanent --add-port=9100/tcp   # node_exporter (if you want remote access)
sudo firewall-cmd --reload
```

**Explanation:**

- `firewall-cmd --permanent` adds rules to the persistent configuration.
- `--reload` applies them without reboot.
- You may restrict sources (e.g., only your admin IP) later for security.

---
