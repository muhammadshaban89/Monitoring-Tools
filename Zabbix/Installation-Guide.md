
#  **ZABBIX SERVER INSTALLATION**

---

# **1. System Update**

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

# **2. Install Zabbix Repository**

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-latest.el9.noarch.rpm
yum clean all
```

###  Why we do this
- Zabbix packages are NOT included in default RHEL/Rocky repos  
- This repo provides official, stable Zabbix packages  
- `yum clean all` refreshes metadata so the repo is recognized  

### 🛠 Troubleshooting
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

# **3. Install Zabbix Server, Frontend, Agent**

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

# **4. Install and Configure MySQL/MariaDB**

```bash
dnf -y install mysql-server
systemctl enable --now mysqld
mysql_secure_installation
```

###  Why we do this
- Zabbix requires a database to store configuration + history  
- `mysql_secure_installation` improves DB security  
- Enabling service ensures DB starts after reboot  

###  Troubleshooting
- **mysqld fails to start:**  
  ```
  journalctl -xeu mysqld
  ```
- **Cannot run mysql_secure_installation:**  
  Ensure DB is running:
  ```
  systemctl start mysqld
  ```

---

# **5. Log into MySQL**

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

# **6. Create Zabbix Database**

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```

###  Why we do this
- Zabbix needs its own dedicated DB  
- `utf8mb4` supports full Unicode  
- `utf8mb4_bin` ensures case-sensitive matching  

###  Troubleshooting
- **Unknown collation:**  
  Your MariaDB version is too old → update DB.

---

# **7. Create Zabbix User + Privileges**

```sql
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'your_actual_password';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'%' identified by 'emaaz@123';
FLUSH PRIVILEGES;
SET GLOBAL log_bin_trust_function_creators = 1;
```

###  Why we do this
- Zabbix server uses this user to connect to DB  
- Grants full access to Zabbix DB  
- `log_bin_trust_function_creators` required for stored functions  

### 🛠 Troubleshooting
- **User not created:**  
  ```
  SELECT user,host FROM mysql.user;
  ```
- **Permission denied:**  
  Re-run GRANT command.

---

# **8. Import Initial Schema**

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix
```

###  Why we do this
- Loads all Zabbix tables, indexes, and default data  
- Without this, Zabbix server cannot start  

### 🛠 Troubleshooting
- **File not found:**  
  ```
  ls /usr/share/zabbix-sql-scripts/mysql/
  ```
- **Import errors:**  
  Ensure DB user has privileges.

---

# **9. Configure Zabbix Server**

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

# **10. Configure PHP for Zabbix Frontend**

```bash
vi /etc/php-fpm.d/zabbix.conf
php_value date.timezone Asia/Karachi
systemctl enable --now httpd
```

###  Why we do this
- Zabbix frontend requires correct timezone  
- Apache must be running to serve UI  

### 🛠 Troubleshooting
- **Timezone warning in UI:**  
  Restart PHP-FPM:
  ```
  systemctl restart php-fpm
  ```

---

# **11. Start Zabbix Server + Agent**

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

# **12. Configure Firewall**

```bash
firewall-cmd --permanent --add-port=10051/tcp
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

### ✅ Why we do this
- Port 10051 → Zabbix server  
- HTTP → Zabbix frontend  

### 🛠 Troubleshooting
- Test port:
  ```
  ss -tulpn | grep 10051
  ```

---

# **13. Configure SELinux**

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

# **14. Access Zabbix Web Interface**

```
http://your_server_ip/zabbix
```

###  Troubleshooting
- **Blank page:**  
  ```
  tail -f /var/log/httpd/error_log
  ```

---

# **15. Login with Default Credentials**

```
Username: Admin
Password: zabbix
```

###  Troubleshooting
- If login fails → clear browser cache or restart Apache.

---

#  **ZABBIX CLIENT INSTALLATION — WITH EXPLANATION + TROUBLESHOOTING**

---

# **1. Set Hostname**

```bash
hostnamectl hostname zabbix.client.local
```

###  Troubleshooting
- Reboot if hostname does not update.

---

# **2. SELinux Handling**

```bash
sed -i 's/^SELINUX=.*/SELINUX=permissive/g' etc/selinux/config && setenforce 0
```

###  Troubleshooting
- Check mode:
  ```
  getenforce
  ```

---

# **3. Add Zabbix Repo**

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-latest.el9.noarch.rpm
yum clean all
```

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

