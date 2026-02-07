**Nagios Host and Client Installation Guide**


- Part 1: **Nagios Server (RHEL 9.5)**
- Part 2: **NRPE Client**
- Part 3: **Server–Client glue (configs)**

---

## Part 1 – Nagios server on RHEL 9.5

### Step 1 — Install required packages

```bash
dnf install -y php perl httpd wget unzip glibc automake glibc-common \
gettext autoconf php-cli gcc gd gd-devel net-snmp openssl-devel \
postfix net-snmp-utils

dnf groupinstall -y "Development Tools"
```
Note : Set selinux to permisive as if  selunix is in enforcing mode ,it  might cause some issues and you have to set booleans .
```bash
sed -i 's/SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config && setenforce 0	
```
**What this does:**

- **php, php-cli** → Needed for Nagios web interface (CGIs and some PHP pages).
- **perl** → Many plugins and scripts are written in Perl.
- **httpd** → Apache web server; Nagios UI is served via Apache.
- **wget, unzip, curl** → Used to download and extract source tarballs.
- **glibc, glibc-common, gettext, autoconf, automake, gcc** → Toolchain and libraries required to compile Nagios and plugins from source.
- **gd, gd-devel** → Graphics library used by some plugins (e.g., graphing, image-related checks).
- **net-snmp, net-snmp-utils** → SNMP tools for monitoring network devices and SNMP-enabled hosts.
- **openssl-devel** → SSL/TLS support for secure communication (NRPE, etc.).
- **postfix** → MTA for sending email alerts.
- **"Development Tools" group** → Installs compilers, make, and other build tools in one shot.

**Why it matters:**  
Nagios Core and its plugins are not just binaries—you’re compiling them. Missing any of these can cause `./configure` or `make` to fail.

---

### Step 2 — Enable Apache

```bash
systemctl enable --now httpd
```

**What this does:**

- **enable** → Apache will start automatically on boot.
- **now** → Starts Apache immediately.

**Why it matters:**  
Nagios’ web interface is accessed via Apache. If Apache isn’t running, you can’t log into Nagios.

---

### Step 3 — Define Nagios version

```bash
export VER="4.4.6"
cd ~
```

**What this does:**

- `export VER="4.4.6"` → Sets a shell variable `VER` so you can reuse it in URLs and paths.
- `cd ~` → Ensures you’re in your home directory (clean workspace).

**Why it matters:**  
Using a variable makes it easy to upgrade later—just change `VER` once instead of editing multiple commands.

---

### Step 4 — Download & extract Nagios Core

```bash
curl -SL https://github.com/NagiosEnterprises/nagioscore/releases/download/nagios-$VER/nagios-$VER.tar.gz | tar -xzvf -
```

**What this does:**

- `curl -SL URL` → Downloads the tarball from GitHub.
- Pipe (`|`) → Sends the downloaded stream directly to `tar`.
- `tar -xvf -` → Extracts from stdin (`-`), verbose mode.

**Why it matters:**  
You get the Nagios Core source code into a directory like `nagios-4.4.6/`.

---

### Step 5 — Configure Nagios

```bash
cd nagios-$VER
./configure
```

**What this does:**

- `./configure` checks your system for required libraries, headers, and tools.
- It generates Makefiles tailored to your environment.

**Why it matters:**  
If something is missing (e.g., `gd-devel`), `./configure` will fail and tell you what to install.

---

### Step 6 — Compile Nagios

```bash
make all
```

**What this does:**

- Compiles:
  - Nagios daemon (`nagios`)
  - CGIs (web interface binaries)
  - Tools and utilities

**Why it matters:**  
This turns the source code into actual binaries that can run on your system.

---

### Step 7 — Create users & groups

```bash
make install-groups-users

groupadd -r nagios
useradd -r -g nagios nagios
usermod -a -G nagios apache
```

**What this does:**

- `make install-groups-users` → Creates default `nagios` user/group if not present (depends on version).
- `groupadd -r nagios` → Creates a system group `nagios`.
- `useradd -r -g nagios nagios` → Creates a system user `nagios` in group `nagios`.
- `usermod -a -G nagios apache` → Adds Apache user to `nagios` group.

**Why it matters:**

- Nagios daemon runs as `nagios` user → isolation and security.
- Apache needs to be in the same group to write to Nagios command file (for web actions like ACK, schedule downtime).

---

### Step 8 — Install Nagios base

```bash
make install
```

**What this does:**

- Installs:
  - Nagios binaries into `/usr/local/nagios/bin`
  - Default config into `/usr/local/nagios/etc`
  - Directory structure (logs, var, etc.)

**Why it matters:**  
This is the core installation step—without it, nothing is in place.

---

### Step 9 — Install systemd service

```bash
make install-daemoninit
```

**What this does:**

- Installs a systemd service file for Nagios (e.g., `/usr/lib/systemd/system/nagios.service`).

**Why it matters:**  
You can manage Nagios with `systemctl start|stop|enable nagios`.

---

### Step 10 — Install command mode permissions

```bash
make install-commandmode
```

**What this does:**

- Sets permissions on the external command file (usually in `/usr/local/nagios/var/rw/nagios.cmd`).
- Ensures `nagios` and group members (like `apache`) can write to it.

**Why it matters:**  
This is how the web UI sends commands to the Nagios daemon (ACKs, reschedule checks, etc.).

---

### Step 11 — Install sample configs

```bash
make install-config
```

**What this does:**

- Installs example config files:
  - `nagios.cfg`
  - `objects/localhost.cfg`
  - `objects/commands.cfg`
  - `objects/contacts.cfg`, etc.

**Why it matters:**  
Gives you a working baseline: Nagios will monitor `localhost` out of the box.

---

### Step 12 — Install Apache web config

```bash
make install-webconf
```

**What this does:**

- Creates an Apache config file (e.g., `/etc/httpd/conf.d/nagios.conf`).
- Defines the `/nagios` URL, CGI settings, auth requirements.

**Why it matters:**  
This is what makes `http://server/nagios` work.

---

### Step 13 — Install Exfoliation theme

```bash
make install-exfoliation
```

**What this does:**

- Installs the “Exfoliation” theme for the Nagios web UI.

**Why it matters:**  
Cosmetic, but nicer than the very old default.

---

### Step 14 — Install Classic UI

```bash
make install-classicui
```

**What this does:**

- Installs the classic Nagios web interface files (HTML, CGI, images).

**Why it matters:**  
This is the actual UI you’ll use to view hosts, services, alerts.

---

### Step 15 — Create Nagios admin user

```bash
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```

**What this does:**

- Creates an HTTP Basic Auth user `nagiosadmin`.
- `-c` creates the file; later users should be added without `-c`.

**Why it matters:**  
You’ll log into the web UI with this username and password.

---

### Step 16 — Restart Apache

```bash
systemctl restart httpd
```

**What this does:**

- Reloads Apache so it picks up the new `nagios.conf` and auth settings.

---

### Step 17–19 — Install Nagios plugins

```bash
export VER="2.4.6"
curl -SL https://github.com/nagios-plugins/nagios-plugins/releases/download/release-$VER/nagios-plugins-$VER.tar.gz \
  | tar -xzvf -
cd nagios-plugins-$VER

./configure --with-nagios-user=nagios --with-nagios-group=nagios
make
make install
```

**What this does:**

- Downloads and extracts official Nagios plugins.
- Configures them to run as `nagios:nagios`.
- Compiles and installs them into `/usr/local/nagios/libexec`.

**Why it matters:**  
Nagios Core alone can’t check anything. Plugins like `check_ping`, `check_http`, `check_disk` are what actually do the monitoring.

---

### Step 20 — Verify Nagios config

```bash
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

**What this does:**

- Validates the entire Nagios configuration.
- Checks for:
  - Syntax errors
  - Missing objects
  - Bad paths
  - Duplicates

**Why it matters:**  
Never restart Nagios blindly—always validate first. This command is your best friend.

---

### Step 21 — Enable & start Nagios

```bash
systemctl enable --now nagios
```

**What this does:**

- `enable` → start at boot.
- `now` → start immediately.

---

### Step 22 — Firewall rules

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

**What this does:**

- Allows HTTP/HTTPS traffic through firewalld.
- `--permanent` → persists across reboots.
- `--reload` → applies changes.

**Why it matters:**  
Without this, you might not reach `http://server/nagios` from your browser.

---

## Part 2 – NRPE client (Linux host to be monitored)
------------------------------------------------------

### Step 1 — Install required packages

```bash
dnf install -y gcc glibc glibc-common openssl openssl-devel perl wget
```

**What this does:**

- Installs compiler, libraries, SSL, Perl, wget.
- Needed to build NRPE from source.

---

### Step 2 — Go to source directory

```bash
cd /usr/src
```

**Why:**  
`/usr/src` is a conventional place to keep source code on Linux.

---

### Step 3 — Download NRPE

```bash
wget https://github.com/NagiosEnterprises/nrpe/releases/download/nrpe-4.0.3/nrpe-4.0.3.tar.gz
tar -xvf nrpe-4.0.3.tar.gz
cd nrpe-4.0.3
```

**What this does:**

- Downloads NRPE source.
- Extracts it.
- Enters the source directory.

---

### Step 4 — Configure & compile NRPE

```bash
./configure --enable-command-args
make all
```

**What this does:**

- `./configure` → prepares build system.
- `--enable-command-args` → allows passing arguments from Nagios to NRPE commands (needed for flexible checks).
- `make all` → builds NRPE daemon and plugins.

**Why it matters:**  
Without `--enable-command-args`, you can’t pass `$ARG1$` etc. from Nagios to NRPE commands.

---

### Step 5 — Create users & groups

```bash
make install-groups-users

groupadd -r nagios
useradd -r -g nagios nagios
```

**What this does:**

- Creates `nagios` user/group on the client.
- NRPE and plugins will run as this user.

**Why it matters:**  
Security isolation—checks don’t run as root.

---

### Step 6 — Install NRPE & config

```bash
make install
make install-config
```

**What this does:**

- Installs NRPE binary (usually `/usr/local/nagios/bin/nrpe`).
- Installs default config file (`/usr/local/nagios/etc/nrpe.cfg`).

---

### Step 7 — Add NRPE to /etc/services

```bash
sh -c "echo >> /etc/services"
sh -c "sudo echo 'Nagios Services' >> /etc/services"
sh -c "sudo echo 'nrpe 5666/tcp' >> /etc/services"
```

**What this does:**

- Registers NRPE service name and port in `/etc/services`.

**Why it matters:**  
Some tools refer to `nrpe` by name; this makes it resolvable to port 5666.

---

### Step 8 — Install systemd service

```bash
make install-init
systemctl enable --now nrpe
```

**What this does:**

- Installs systemd service for NRPE.
- Enables and starts NRPE daemon.

**Why it matters:**  
NRPE must be running and listening on port 5666 for Nagios to connect.

---

### Step 9 — Edit NRPE config

```bash
vi /usr/local/nagios/etc/nrpe.cfg
```

Key settings:

```ini
allowed_hosts=<Nagios_Server_IP>
dont_blame_nrpe=1
```

- **allowed_hosts** → Only this IP (Nagios server) can talk to NRPE.
- **dont_blame_nrpe=1** → Allows passing arguments from Nagios (needed for flexible commands).

- Then Uncomment all:

**MISC System matric**


You also define commands like:

- If you want to monitor HTTP, HTTPS, FTP, Disk Free, and Memory Utilization for a Linux client in Nagios, you need to add two things:
1. 	NRPE commands on the client (so the client knows what to execute)
2. 	Service definitions on the Nagios server (so Nagios knows what to check)

You do NOT add these in one place.

You add commands on the client and services on the server.

```ini
command[check_root]=/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /
command[check_disk_home]=/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /home
command[check_http]=/usr/local/nagios/libexec/check_http -I 127.0.0.1
command[check_https]=/usr/local/nagios/libexec/check_http -I 127.0.0.1 -S
command[check_mem]=/usr/local/nagios/libexec/check_mem -w 80 -c 90


```

**Why it matters:**  
This is where you define what NRPE can execute. Nagios will call these by name.

---

### Step 10 — Open firewall port

```bash
firewall-cmd --permanent --add-port=5666/tcp
firewall-cmd --reload
```

**What this does:**

- Allows NRPE traffic from Nagios server to client on port 5666.

---

## Part 3 – Glue on the server (host + service definitions)  -At server Side 

### Step 11 — Create servers directory

```bash
mkdir /usr/local/nagios/etc/servers
chown -R nagios:nagios /usr/local/nagios/etc/servers
chmod g+w /usr/local/nagios/etc/servers
```

**What this does:**

- Creates a dedicated directory for per-host configs.
- Sets ownership and group write permissions.

**Why it matters:**  
Keeps configs organized—each client gets its own `.cfg` file.

---

### Step 12 — Enable directory in nagios.cfg

```bash
vi /usr/local/nagios/etc/nagios.cfg
```

Uncomment:

```ini
cfg_dir=/usr/local/nagios/etc/servers
```

**What this does:**

- Tells Nagios to load all `.cfg` files from that directory.

---

### Step 13 — Add NRPE command
- Define modules here  i.e things to monitor.

```bash
vi /usr/local/nagios/etc/objects/commands.cfg
```

Add at end of this file:

```cfg
define command{
    command_name    check_nrpe
    command_line    $USER1$/check_nrpe -H $HOSTADDRESS$ -t 30 -c $ARG1$
}


```
- This is the only command you need on the server.
- Everything else is handled by NRPE on the client.
  
**What this does:**

- Defines a reusable command `check_nrpe`.
- Each part has a purpose:

| Part | Meaning |
|------|---------|
| `$USER1$` | Path to Nagios plugins directory (usually `/usr/local/nagios/libexec`) |
| `check_nrpe` | The plugin that talks to NRPE on the client |
| `-H $HOSTADDRESS$` | IP of the client being monitored |
| `-t 30` | Timeout (30 seconds) |
| `-c $ARG1$` | The NRPE command name defined on the client |


**Why it matters:**  
This is the bridge between Nagios and NRPE.

---

### Step 14 — Create host config

```bash
vi /usr/local/nagios/etc/servers/yourhost.cfg

```

Add:

```cfg
define host{
    use                     linux-server
    host_name               primary.server.local
    alias                   My Client Server
    address                 192.168.100.19
    max_check_attempts      5
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}

#For Ping
define service{
    use                     generic-service
    host_name               primary.server.local
    service_description     PING
    check_command           check_ping!200.0,20%!400.0,90%
#check_ping!warning_threshold!critical_threshold
}
#For HTTP
define service{
    use                     generic-service
    host_name               primary.server.local
    service_description     HTTP Service
    check_command           check_nrpe!check_http
}
#For HTTPS
define service{
    use                     generic-service
    host_name               primary.server.local
    service_description     HTTPS Service
    check_command           check_nrpe!check_https
}
#For FTP
define service{
    use                     generic-service
    host_name               primary.server.local
    service_description     FTP Service
    check_command           check_nrpe!check_ftp
}
#For Disk Free 
define service{
    use                     generic-service
    host_name               primary.server.local
    service_description     Disk Free /
    check_command           check_nrpe!check_disk_root
}

#For Memory Utilization
define service{
    use                     generic-service
    host_name               primary.server.local
    service_description     Memory Utilization
    check_command           check_nrpe!check_mem
}

```

**What this does:**

- **Host block** → Defines the client machine:
  - `host_name` → logical name used in Nagios.
  - `address` → IP address of the client.
- **Service A service block defines the thing you want to check on that machine
  - `check_command check_nrpe!check_root`
  - `check_root` must exist in `nrpe.cfg` on the client.
- **Service PING** → Uses local `check_ping` plugin from Nagios server.

**Why it matters:**  
This is where you actually tell Nagios: “Monitor this host and these services.”

---

### Step 15 — Restart Nagios

```bash
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
systemctl restart nagios
```

**What this does:**

- Validates config.
- Restarts Nagios to apply changes.

---

After that You can open browser  : `http//ip/nagios`  ,it will open Nagios Web-UI.
----
