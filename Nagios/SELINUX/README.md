
SELINUX 
-------

- SELinux, when enforcing, **blocks** many of these actions unless you explicitly allow them.

- steps fix the most common SELinux denials.

---

# **8.1 Install SELinux Tools**

```
dnf install -y policycoreutils-python-utils
```

### **Purpose**
This package gives you tools like:

- `semanage`
- `setsebool`
- `restorecon`
- `audit2allow`

These are required to manage SELinux policies and troubleshoot denials.

Without this package, you cannot modify SELinux booleans or inspect AVC logs properly.

---

# **8.2 Allow Apache to Connect to Nagios and Run CGIs**

### **Command 1**
```
setsebool -P httpd_can_network_connect 1
```

### **What it does**
Allows Apache (httpd) to make **outbound network connections**.

### **Why Nagios needs this**
Nagios web interface uses CGI scripts that may:

- Contact NRPE  
- Contact remote hosts  
- Perform HTTP checks  
- Run plugins that need network access  

Without this boolean, Apache cannot connect to anything outside the server, and you get SELinux denials.

---

### **Command 2**
```
setsebool -P httpd_unified 1
```

### **What it does**
Allows Apache to:

- Read/write files in certain directories  
- Access Nagios command pipe  
- Run CGI scripts more freely  

### **Why Nagios needs this**
Nagios uses:

```
/usr/local/nagios/var/rw/nagios.cmd
```

This is the **external command file** used for:

- Acknowledging alerts  
- Scheduling downtime  
- Forcing checks  
- Submitting passive results  

Apache must be able to write to this file.  
SELinux blocks it unless `httpd_unified` is enabled.

---

# **When do you need these booleans?**

You need them when:

- SELinux is **enforcing**  
- You want Nagios to run normally  
- You want to avoid switching SELinux to permissive mode  
- You want Apache to run Nagios CGIs without errors  

If you skip these, you will see errors like:

- “Error: Could not read status file”  
- “Could not open command file”  
- “403 Forbidden”  
- “Unable to connect to remote host”  
- “NRPE: Socket timeout”  

---

# **How to Inspect SELinux Denials**

```
ausearch -m avc -ts recent
```

### **What this does**
Shows the most recent SELinux AVC (Access Vector Cache) denials.

This is extremely useful when:

- Nagios CGIs fail  
- Apache cannot write to nagios.cmd  
- NRPE checks fail  
- HTTP/HTTPS checks fail  
- Plugins cannot access files  

You can then use:

```
audit2allow -w -a
```

to see human-readable explanations.

---

# **Should you keep SELinux enforcing?**

If you want maximum security: **Yes**  
If you want simplicity: **Permissive is easier**

But with the booleans we listed, Nagios works perfectly in enforcing mode.

---

