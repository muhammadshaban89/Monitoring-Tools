
#  **ZABBIX SERVER INSTALLATION**

---

#### **1. Set hostname**

```bash
hostnamectl set-hostname FQDN
```

**What this does:**  
This sets the **permanent hostname** of the server (e.g. `zabbix.lab.local`). The hostname is used by:

- Zabbix frontend (it may show in UI and logs)  
- SSL certificates (if you use HTTPS later)  
- DNS resolution and general identification  

**Why it matters:**  
A stable, meaningful hostname avoids confusion when you have multiple monitored systems and makes logs, alerts, and dashboards easier to understand.

---

#### **2. SELinux mode**

```bash
sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config && setenforce 0
```

- or enable booleans
```
setsebool -P  httpd_can_network_connect on
```


**What this does:**

- Edits `/etc/selinux/config` so that after reboot, SELinux will be in **permissive** mode.
- `setenforce 0` switches the current running mode from **enforcing** to **permissive** immediately.

**Why it matters:**  
Zabbix involves multiple components (Apache, PHP, MariaDB, Zabbix server, agent). SELinux in enforcing mode can silently block some of their interactions unless you configure proper booleans and contexts. Setting it to permissive during initial setup avoids SELinux‑related issues while you’re still building and learning the stack.



# **3. System Update**

```bash
dnf update -y
```

###  Why we do this
- Ensures all system packages are up to date  
- Prevents dependency conflicts during Zabbix installation  
- Applies security patches and bug fixes  

###  Troubleshooting
- **Update fails:**  
  Check internet connectivity:
  ```
  ping 8.8.8.8
  ```
- **Repo errors:**  
  ```
  dnf clean all
  dnf makecache
  ```

---

# **4. Install Zabbix Repository**

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-latest.el9.noarch.rpm
yum clean all
```

###  Why we do this
- Zabbix packages are NOT included in default RHEL/Rocky repos  
- This repo provides official, stable Zabbix packages  
- `yum clean all` refreshes metadata so the repo is recognized  

###  Troubleshooting
- **Repo not added:**  
  ```
  ls /etc/yum.repos.d | grep zabbix
  ```
- **SSL or download errors:**  
  Check DNS:
  ```
  ping repo.zabbix.com
  ```

---

# **5. Install Zabbix Server, Frontend, Agent**

```bash
dnf install zabbix-server-mysql zabbix-sql-scripts zabbix-web-mysql zabbix-apache-conf zabbix-agent -y
```

###  Why we do this
- **zabbix-server-mysql** → main backend engine  
- **zabbix-sql-scripts** → DB schema files  
- **zabbix-web-mysql** → PHP frontend  
- **zabbix-apache-conf** → Apache config for UI  
- **zabbix-agent** → monitor the Zabbix server itself  

### Troubleshooting
- **Package conflicts:**  
  Disable EPEL:
  ```
  dnf config-manager --set-disabled epel
  ```
- **Missing dependencies:**  
  ```
  dnf clean all
  dnf install <package>
  ```

---

# **6. Install and Configure MySQL/MariaDB**

```bash
yum install mariadb* -y
systemctl enable --now mariadb
systemctl status mariadb
mysql_secure_installation

```

###  Why we do this
- Zabbix requires a database to store configuration + history  
- `mysql_secure_installation` improves DB security  
- Enabling service ensures DB starts after reboot  

###  Troubleshooting
- **mysqld fails to start:**  
  ```
  journalctl -xeu mariadb
  ```
- **Cannot run mysql_secure_installation:**  
  Ensure DB is running:
  ```
  systemctl start mariadb
  ```

---

# **7. Log into MySQL**

```bash
mysql -u root -p
```

###   Why we do this
- Needed to create Zabbix database and user  
- Root access required for DB creation  

###  Troubleshooting
- **Access denied:**  
  Wrong password → rerun:
  ```
  mysql_secure_installation
  ```

---

# **8. Create Zabbix Database**

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```
- Show databases:
```
SHOW DATABASES;
```
###  Why we do this
- Zabbix needs its own dedicated DB  
- `utf8mb4` supports full Unicode  
- `utf8mb4_bin` ensures case-sensitive matching  

###  Troubleshooting
- **Unknown collation:**  
  Your MariaDB version is too old → update DB.

---

# **9. Create Zabbix User + Privileges**

```sql
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'your_actual_password';
CREATE USER 'zabbix'@'%' IDENTIFIED BY 'your_actual_password';

GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'%';

FLUSH PRIVILEGES;
SET GLOBAL log_bin_trust_function_creators = 1;
```

###  Why we do this
- Zabbix server uses this user to connect to DB  
- Grants full access to Zabbix DB  
- `log_bin_trust_function_creators` required for stored functions  

###  Troubleshooting
- **User not created:**  
  ```
  SELECT user,host FROM mysql.user;
  ```
- **Permission denied:**  
  Re-run GRANT command.

---

# **10. Import Initial Schema**

```bash
mysql -uroot -p'Emaaz@123' -e "SET GLOBAL innodb_strict_mode='OFF';"

zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p zabbix

mysql -uroot -p'Emaaz@123' -e "SET GLOBAL innodb_strict_mode='ON';"
```
###  Why we do this
- Loads all Zabbix tables, indexes, and default data  
- Without this, Zabbix server cannot start
- Some MySQL/MariaDB versions enforce strict rules on:
- index lengths
- row formats
- foreign key constraints
- default values
  
The Zabbix schema (especially older versions) contains:
- long index names
- large VARCHAR fields
- stored functions
- triggers
These may fail when innodb_strict_mode=ON.
So the import may break with errors

###  Troubleshooting
- **File not found:**  
  ```
  ls /usr/share/zabbix-sql-scripts/mysql/
  ```
- **Import errors:**  
  Ensure DB user has privileges.

---

# **11. Configure Zabbix Server**

```bash
vim /etc/zabbix/zabbix_server.conf
```

Set:

```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=your_actual_password
NodeAddress=localhost:10051
```

###  Why we do this
- Tells Zabbix server how to connect to the database  
- Wrong DB credentials = server won’t start  

###  Troubleshooting
- Check logs:
  ```
  tail -f /var/log/zabbix/zabbix_server.log
  ```

---

# **12. Configure PHP for Zabbix Frontend**

```bash
vi /etc/php-fpm.d/zabbix.conf
```
- Then add:
```
php_value[date.timezone] = Asia/Riyadh
systemctl enable --now httpd
```

###  Why we do this
- Zabbix frontend requires correct timezone  
- Apache must be running to serve UI  

###  Troubleshooting
- **Timezone warning in UI:**  
  Restart PHP-FPM:
  ```
  systemctl restart php-fpm
  ```

---

# **13. Start Zabbix Server + Agent**

```bash
systemctl enable --now zabbix-server zabbix-agent
```

###  Why we do this
- Starts Zabbix backend  
- Starts agent to monitor the server itself  

###  Troubleshooting
- **Service failed:**  
  ```
  systemctl status zabbix-server
  ```

---

# **14. Configure Firewall**

```bash
firewall-cmd --permanent --add-port=10051/tcp
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

###  Why we do this
- Port 10051 → Zabbix server  
- HTTP → Zabbix frontend  

###  Troubleshooting
- Test port:
  ```
  ss -tulpn | grep 10051
  ```

---

# **15. Configure SELinux**

```bash
setsebool -P httpd_can_network_connect on
```

###  Why we do this
- Allows Apache/PHP to communicate with Zabbix server  
- Required when SELinux is enforcing  

###  Troubleshooting
- Check SELinux mode:
  ```
  getenforce
  ```

---

# **16. Access Zabbix Web Interface**

```
http://your_server_ip/zabbix
```

###  Troubleshooting
- **Blank page:**  
  ```
  tail -f /var/log/httpd/error_log
  ```

---

# **17. Login with Default Credentials**

```
Username: Admin
Password: zabbix
```

###  Troubleshooting
- If login fails → clear browser cache or restart Apache.

---

#  **ZABBIX CLIENT INSTALLATION — WITH EXPLANATION + TROUBLESHOOTING**

---
#### **1. Set hostname**

```bash
hostnamectl hostname zabbix.client.local
```

**What this does:**

- Sets the hostname of the client machine to something meaningful.

**Why it matters:**  
This hostname is what you’ll see in the Zabbix UI (if you configure the agent to use it). It helps you identify which machine is which when you have many clients.

---

#### **2. SELinux handling on client**

```bash
sed -i 's/^SELINUX=.*/SELINUX=permissive/g' etc/selinux/config && setenforce 0
```
- or enable booleans
```
setsebool -P  httpd_can_network_connect on
```

**What this does:**

- Either sets SELinux to permissive (less strict) or enables specific SELinux booleans to allow network connections.

**Why it matters:**  
On a client, SELinux can block the agent from accepting connections or sending data, depending on configuration. Relaxing it or setting proper booleans avoids silent failures.




# **3. Add Zabbix Repo**

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-latest.el9.noarch.rpm
yum clean all
```

- if client is centos7
```
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/7/x86_64/zabbix-release-latest.el7.noarch.rpm
yum clean all
````

###  Troubleshooting
- Check repo:
  ```
  dnf repolist
  ```

---

# **4. Install Zabbix Agent**

```bash
dnf install zabbix-agent
```

###  Troubleshooting
- If package not found → repo missing.

---

# **5. Configure Agent**

```bash
vi /etc/zabbix/zabbix_agentd.conf
Server=192.168.100.20
ServerActive=192.168.100.20
HostName=zabbix
```

###  Troubleshooting
- Check logs:
  ```
  tail -f /var/log/zabbix/zabbix_agentd.log
  ```

---

# **6. Start Agent**

```bash
systemctl restart zabbix-agent
systemctl enable --now zabbix-agent
```

###  Troubleshooting
- Check status:
  ```
  systemctl status zabbix-agent
  ```

---

# **7. Firewall**

```bash
firewall-cmd --add-port=10050/tcp --permanent
firewall-cmd --reload
```

###  Troubleshooting
- Test from server:
  ```
  telnet <client-ip> 10050
  ```

---

# **8. Add Client in Zabbix Dashboard**

###  Troubleshooting
- Host shows “Unavailable”:  
  - Wrong hostname  
  - Wrong IP  
  - Template not linked  
  - Agent not running  

---

