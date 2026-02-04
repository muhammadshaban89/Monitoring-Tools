**PRTG Network Monitor**
------------------------

- is an all‑in‑one, agentless monitoring platform that tracks the health, performance, and availability of your entire IT infrastructure.
-  It works by using “sensors” — small monitoring units — to collect data from devices, servers, applications, and cloud services.
- It provides real‑time dashboards, alerts, and reports to help IT teams detect issues early and maintain uptime.

- Monitors devices using **SNMP, WMI, Ping, HTTP, SQL, Flow protocols, and cloud APIs**.  
- Offers a **single pane of glass** for your entire infrastructure.  
- Designed for **small, medium, and large enterprises**.  


---

# How PRTG Works
PRTG operates using a **core server** and optional **remote probes**.

# 1. **Core Server**
The core server handles:
- Configuration  
- Data processing  
- Web interface  
- Notifications  
- Reporting  

# 2. **Probes**
Probes are responsible for collecting monitoring data.

**Local Probe**
- Installed automatically with the core server.  
- It performs monitoring inside the same network.
**Remote Probes**
Used for:
- Distributed monitoring  
- Monitoring remote sites  
- Monitoring DMZ networks  

# 3. **Sensors**
Sensors are the smallest monitoring units.  
Each sensor monitors **one specific metric** (e.g., CPU load, disk usage, ping time).

PRTG collects data through:
- SNMP  
- WMI  
- SSH  
- HTTP/HTTPS  
- SQL queries  
- Cloud APIs (AWS, Azure, Google Cloud)  
- Packet sniffing  
- Flow protocols (NetFlow, sFlow, jFlow)  


---

# Basic System Requirements for PRTG
PRTG can be installed **on‑premises** or used as **PRTG Hosted**.

# ✔️ **Operating System**
- Windows Server 2016, 2019, 2022  
- Windows 10/11 (for small deployments/testing)

# ✔️ **Hardware Requirements**  
(Depends on number of sensors)

| Deployment Size | CPU | RAM | Disk |
|-----------------|-----|-----|------|
| Small (<1,000 sensors) | 2–4 cores | 8–16 GB | SSD recommended |
| Medium (1,000–5,000 sensors) | 4–8 cores | 16–32 GB | SSD strongly recommended |
| Large (5,000–10,000+ sensors) | 8–16 cores | 32–64 GB | High‑performance SSD |

PRTG documentation includes detailed system requirements.  


# ✔️ **Network Requirements**
- Stable LAN/WAN  
- SNMP enabled on devices  
- Firewall rules allowing probe communication  
- Optional: Cloud API access for AWS/Azure monitoring  

---

#  What Are PRTG Sensors?
Sensors are the **core building blocks** of PRTG monitoring.

# **Definition**
A sensor = **one monitoring metric**  
Example:  
- 1 Ping Sensor = monitors ping time  
- 1 CPU Load Sensor = monitors CPU usage  
- 1 HTTP Sensor = monitors website availability  

PRTG includes **hundreds of sensor types**, such as:
- **Ping Sensor** – checks device availability  
- **SNMP Traffic Sensor** – monitors bandwidth  
- **WMI CPU Sensor** – monitors Windows CPU  
- **HTTP Sensor** – checks website response  
- **SQL Sensor** – runs SQL queries  
- **AWS CloudWatch Sensors** – monitor AWS EC2, RDS, EBS, etc.  
- **VMware Sensors** – ESXi host, VM performance  
- **DNS Sensor** – checks DNS resolution  


Sensors are grouped into categories:
- **Network sensors** (SNMP, Ping, Flow)  
- **Server sensors** (WMI, SSH, VMware)  
- **Application sensors** (SQL, HTTP, Exchange, AD)  
- **Cloud sensors** (AWS, Azure, Google Cloud)  
- **Hardware sensors** (Dell, HP, Cisco, etc.)  

---

# How PRTG Alerts & Dashboards Work
PRTG provides:
- Real‑time alerts (email, SMS, push notifications)  
- Custom thresholds  
- Maps & dashboards  
- Reports (daily, weekly, monthly)  
- Preconfigured templates for common devices  


---

# Summary

- PRTG is a powerful, flexible monitoring platform that uses sensors to track every component of your IT infrastructure. 
- It works through a core server and probes, collecting data via SNMP, WMI, HTTP, cloud APIs, and more.
- With hundreds of sensor types, customizable dashboards, and real‑time alerts, it helps ensure uptime and performance across networks, servers, applications, and cloud environments.


