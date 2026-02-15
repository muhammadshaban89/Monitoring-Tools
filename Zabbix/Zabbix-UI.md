**complete, clean, step‑by‑step guide** that covers:**
--------------------------------------

- How to open the Zabbix UI  
- How to add a new client (agent)  
- How to attach templates for **PING**, **SSH**, and **HTTP**  
- How to verify everything is working  


---

# ⭐ 1. How to Open the Zabbix UI

### On your browser:
Enter the URL of your Zabbix server:

```
http://<ZABBIX_SERVER_IP>/zabbix
```

Example:

```
http://192.168.244.133/zabbix
```

### Login using:
- **Username:** Admin  
- **Password:** zabbix  
(Unless you changed it)

You will land on the **Dashboard**.

---

# ⭐ 2. How to Add a New Client (Zabbix Agent)

### Step 1 — Go to Hosts
Left menu:

**Configuration → Hosts**

### Step 2 — Click “Create host”
Top-right button.

### Step 3 — Fill in Host Details

#### **Host name**
This must match the agent config on the client:

```
Hostname=Centos-Client
```

So in UI:

```
Host name: Centos-Client
```

#### **Groups**
Choose:

```
Linux servers
```

(or any group you prefer)

#### **Interfaces**
Click **Add** → choose **Agent**

Fill in:

- **IP address:** 192.168.244.168  
- **Port:** 10050  

Click **Add**.

---

# ⭐ 3. How to Attach Templates (PING, SSH, HTTP, Linux Monitoring)

Now go to the **Templates** tab inside the same host.

Click **Select** and add these templates:

---

## ✔ 1. Linux Monitoring (required for green ZBX)
```
Linux by Zabbix agent
```

This enables CPU, RAM, Disk, Network, Processes, etc.

---

## ✔ 2. PING Monitoring
```
ICMP Ping
```

This checks:
- Host up/down  
- Latency  
- Packet loss  

---

## ✔ 3. SSH Service Monitoring
```
SSH Service
```

This checks:
- Port 22  
- SSH availability  

---

## ✔ 4. HTTP Service Monitoring
```
HTTP Service
```

This checks:
- Port 80/443  
- Web service availability  

---

## ✔ 5. OS Processes (optional but useful)
```
OS processes by Zabbix agent
```

This monitors:
- Total processes  
- Running/sleeping/zombie  
- Process count by name  

---

### After selecting templates → click **Update**

---

# ⭐ 4. Restart the Agent on the Client

On the client machine:

```bash
systemctl restart zabbix-agent
```

---

# ⭐ 5. Verify Monitoring

### Go to:
**Monitoring → Hosts**

You should now see:

### 🟢 ZBX (green) for Centos‑Client

This means:

- Zabbix server can reach the agent  
- Agent items are being collected  

### Go to:
**Monitoring → Latest data → Select Centos‑Client**

You will now see:

- CPU usage  
- Memory  
- Disk  
- Network  
- Ping  
- SSH  
- HTTP  
- Processes  

Everything will start filling in automatically.

---

# ⭐ 6. Troubleshooting (if ZBX is still grey)

You already confirmed:

- Agent is running  
- Port 10050 is open  
- Hostname matches  
- Server can reach client (telnet OK)

So the ONLY reason it stays grey is:

### ❗ No agent-based template was attached  
(“Linux by Zabbix agent” is required)

Once you attach it → ZBX turns green.

---
