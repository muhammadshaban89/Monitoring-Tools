
### In this section we will discuss how Grafana,Node Exporter and  Prometheus Works

#  **1. Node Exporter — “The Data Collector”**

### **What it is**
A lightweight agent installed on Linux servers (RHEL, CentOS, Ubuntu, etc.).

### **What it does**
- Reads system metrics directly from the OS:
  - CPU usage  
  - RAM usage  
  - Disk I/O  
  - Filesystem usage  
  - Network traffic  
  - Load average  
  - Hardware stats  
- Exposes these metrics on an HTTP endpoint:

```
http://<server-ip>:9100/metrics
```

### **How it works**
- It does NOT push data anywhere.
- It simply **waits** for Prometheus to come and collect the metrics.

### **Think of it like:**
A sensor installed on each machine, exposing raw data.

---

#  **2. Prometheus — “The Time‑Series Database + Scraper”**

### **What it is**
A monitoring system that:
- Scrapes metrics from exporters  
- Stores them in a time‑series database  
- Provides a query language (PromQL)  
- Acts as the data source for Grafana  

### **What it does**
- Reads metrics from Node Exporter (Linux)
- Reads metrics from windows_exporter (Windows)
- Stores all metrics with timestamps
- Evaluates alert rules (if Alertmanager is added)
- Exposes its own UI on:

```
http://<prometheus-ip>:9090
```

### **How it works**
Prometheus **pulls** data from exporters using the `scrape_configs` section in `prometheus.yml`.

Example:

```yaml
scrape_configs:
  - job_name: "centos"
    static_configs:
      - targets: ["192.168.1.10:9100"]
```

Prometheus connects to the target every 15 seconds (default) and stores the metrics.

### **Think of it like:**
A central monitoring brain that:
- Collects  
- Stores  
- Organizes  
- Queries  

all metrics from your infrastructure.

---

#  **3. Grafana — “The Visualization Layer”**

### **What it is**
A dashboard and visualization platform.

### **What it does**
- Connects to Prometheus as a data source
- Reads metrics using PromQL queries
- Displays them in:
  - Graphs  
  - Gauges  
  - Tables  
  - Alerts  
  - Dashboards  

### **How it works**
Grafana does NOT store metrics.  
It only **queries** Prometheus when you open a dashboard.

You configure Prometheus as a data source:

```
http://<prometheus-ip>:9090
```

Then import dashboards like:
- Node Exporter Full (ID: 1860)
- Windows Exporter Dashboard

### **Think of it like:**
A beautiful UI that turns Prometheus data into readable dashboards.

---

#  **4. How They Work Together (Data Flow)**

Here’s the full pipeline:

```
[Linux Server] --node_exporter--> 
                     |
                     v
[Windows Server] --windows_exporter-->
                     |
                     v
            [Prometheus Server]
                     |
                     v
               [Grafana UI]
```

### Step-by-step flow:

1. **Node Exporter** exposes metrics on port **9100**  
2. **windows_exporter** exposes metrics on port **9182**  
3. **Prometheus** scrapes these endpoints  
4. Prometheus stores the metrics in its time-series database  
5. **Grafana** queries Prometheus  
6. Grafana displays dashboards and graphs  

---

#  **5. Why This Architecture Is So Good**

###  Pull-based (Prometheus pulls data)
- No agents pushing data randomly  
- Prometheus controls the schedule  
- More secure and predictable  

###  Lightweight exporters
- node_exporter uses almost no CPU/RAM  
- windows_exporter is also minimal  

###  Grafana is flexible
- Supports alerts  
- Supports multiple data sources  
- Supports custom dashboards  

###  Works across OS types
- Linux  
- Windows  
- Containers  
- Kubernetes  

---



# **6. Summary (Simple Version)**

| Component | Role | Runs On | Port |
|----------|------|---------|------|
| **node_exporter** | Collects Linux metrics | RHEL, CentOS | 9100 |
| **windows_exporter** | Collects Windows metrics | Windows Server | 9182 |
| **Prometheus** | Scrapes + stores metrics | RHEL | 9090 |
| **Grafana** | Visualizes metrics | RHEL | 3000 |

---
