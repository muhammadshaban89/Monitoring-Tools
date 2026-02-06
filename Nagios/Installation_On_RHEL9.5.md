Nagios Installation on RHEL:
----------------------------

I will assume :

- Fresh RHEL 9.5 server  
- You have root (or sudo)  
- SELinux is enforcing  

---

## 1. Prepare the system

### 1.1 Update packages

```bash
dnf update -y
```

**Why:**  
Brings system packages to the latest version—avoids weird build/runtime issues caused by outdated libs.

---

### 1.2 Install required dependencies

```bash
dnf install -y gcc glibc glibc-common wget unzip httpd php php-cli \
gd gd-devel perl make automake openssl openssl-devel \
net-snmp net-snmp-utils mailx
```

**Why each group matters:**

- **gcc, make, automake** → needed to compile Nagios and plugins from source  
- **glibc, openssl, gd, their -devel variants** → libraries Nagios and plugins link against  
- **httpd, php, php-cli** → Apache + PHP for the Nagios web interface  
- **perl** → many plugins are Perl-based  
- **net-snmp, net-snmp-utils** → SNMP checks for network devices  
- **mailx** → for email notifications  

---

## 2. Create Nagios user and groups

```bash
useradd nagios
groupadd nagcmd
usermod -a -G nagcmd nagios
usermod -a -G nagcmd apache
```

**Why:**

- **nagios user** → runs the Nagios daemon (security isolation)  
- **nagcmd group** → controls who can submit external commands (like ACKs, scheduling checks)  
- **apache in nagcmd** → allows the web UI (Apache) to send commands to Nagios (e.g., via web interface actions)  

---

## 3. Download Nagios Core

Pick a stable version (example: 4.4.14; adjust if newer exists).

```bash
cd /tmp
wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.4.14.tar.gz
tar -zxvf nagios-4.4.14.tar.gz
cd nagios-4.4.14
```

**Why:**

- We build from source to stay close to upstream and avoid repo lag.  
- `/tmp` is a safe scratch area for building.  

---

## 4. Configure, compile, and install Nagios Core

### 4.1 Configure

```bash
./configure --with-command-group=nagcmd
```

**Why:**

- `./configure` checks your system for required libs and prepares Makefiles.  
- `--with-command-group=nagcmd` tells Nagios that external commands (like from web UI) will be owned by group `nagcmd`.

---

### 4.2 Compile everything

```bash
make all
```

**Why:**  
Builds:

- Nagios daemon  
- CGIs (web interface binaries)  
- Event handlers and tools  

---

### 4.3 Install core components

```bash
make install
make install-init
make install-commandmode
make install-config
make install-webconf
```

**What each does:**

- **`make install`** → installs Nagios binaries, default config, directory structure under `/usr/local/nagios`  
- **`make install-init`** → installs systemd service file for Nagios  
- **`make install-commandmode`** → sets permissions on the external command file so `nagcmd` group can write to it  
- **`make install-config`** → drops sample config files (hosts, services, commands)  
- **`make install-webconf`** → creates Apache config for `/nagios` URL  

---

## 5. Set up web authentication

Nagios web UI uses HTTP basic auth.

```bash
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```

**Why:**

- Creates a password file and user `nagiosadmin`  
- This is the login you’ll use in the browser  
- `-c` creates the file (omit `-c` if adding more users later)  

---

## 6. Install Nagios Plugins

Nagios Core alone can’t check much—it needs plugins.

```bash
cd /tmp
wget https://nagios-plugins.org/download/nagios-plugins-2.4.6.tar.gz
tar -zxvf nagios-plugins-2.4.6.tar.gz
cd nagios-plugins-2.4.6
./configure --with-nagios-user=nagios --with-nagios-group=nagios
make
make install
```

**Why:**

- Plugins provide checks like `check_ping`, `check_http`, `check_disk`, etc.  
- `--with-nagios-user/group` ensures correct ownership so Nagios can execute them.  

Plugins are usually installed under:

```bash
/usr/local/nagios/libexec
```

---

## 7. Configure Apache for Nagios

### 7.1 Enable and start Apache

```bash
systemctl enable httpd
systemctl start httpd
```

**Why:**  
Nagios web UI is served via Apache; it must be running and enabled at boot.

---

### 7.2 Allow HTTP in firewalld

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

**Why:**  
RHEL 9.5 uses firewalld; you must explicitly allow HTTP traffic so you can reach the web UI.

---

## 8. Handle SELinux (important on RHEL 9.5)

If SELinux is enforcing, Apache needs permission to run Nagios CGIs and access its files.

### 8.1 Install SELinux tools

```bash
dnf install -y policycoreutils-python-utils
```

### 8.2 Allow Apache to connect to Nagios and run CGIs

```bash
setsebool -P httpd_can_network_connect 1
setsebool -P httpd_unified 1
```

**Why:**

- `httpd_can_network_connect` → allows Apache to make outbound connections (useful for some checks)  
- `httpd_unified` → simplifies file access for Apache under SELinux  

If you hit SELinux denials, you can inspect with:

```bash
ausearch -m avc -ts recent
```

and then decide if you need custom policies—but often the above booleans are enough for a lab.

---

## 9. Enable and start Nagios

```bash
systemctl enable nagios
systemctl start nagios
```

**Why:**

- `enable` → start at boot  
- `start` → launch now  

Check status:

```bash
systemctl status nagios
```

If there’s an error, run:

```bash
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

**Why:**  
This validates the config and prints detailed errors (missing `}` , bad paths, etc.).

---

## 10. Access the Nagios Web Interface

From your browser:

```text
http://<RHEL9.5-IP>/nagios
```

Login:

- **Username:** `nagiosadmin`  
- **Password:** the one you set with `htpasswd`  

You should see:

- Hosts: `localhost`  
- Services: ping, load, disk, etc.  

That confirms Nagios is working.

---

## 11. Basic config understanding (so you’re not blind)

Main config file:

```bash
/usr/local/nagios/etc/nagios.cfg
```

Objects (hosts, services, commands):

```bash
/usr/local/nagios/etc/objects/
```

Key files:

- `commands.cfg` → defines what checks do  
- `contacts.cfg` → who gets alerts  
- `localhost.cfg` → example host/service definitions  

When you add new hosts/services, you either:

- Add new `.cfg` files and include them in `nagios.cfg`, or  
- Extend existing ones like `localhost.cfg` (for lab only)  

Always validate after changes:

```bash
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
systemctl reload nagios
```

---

Quick Summary For Nagios Ports:
----

Nagios Web UI
- 80 / 443

Linux Monitoring
- NRPE → 5666
- SSH → 22
- SNMP → 161/162

Windows Monitoring
- NSClient++ (NRPE) → 5666
- NSClient++ (Legacy) → 12489
- WMI → 5985/5986

Cloud / Remote Sites
- NRDP → 80/443
-------------

