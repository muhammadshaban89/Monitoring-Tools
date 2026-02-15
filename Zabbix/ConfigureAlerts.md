
1. How to open the Zabbix UI  
2. How to configure email (SMTP)  
3. How to create a media type  
4. How to assign email to a user  
5. How to enable actions (alerts)  
6. How to test notifications  



---

# ⭐ 1. Open the Zabbix UI

On your browser:

```
http://<ZABBIX_SERVER_IP>/zabbix
```

Example:

```
http://192.168.244.133/zabbix
```

Login:

- **Username:** Admin  
- **Password:** zabbix  

You will land on the **Dashboard**.

---

# ⭐ 2. Configure Email (SMTP) in Zabbix

This is the MOST important part.  
Zabbix must know how to send emails.

### Go to:
**Administration → Media types**

### Find:
```
Email
```

Click **Email** → **Edit**.

### Fill in SMTP details:

Example for Gmail (if you use Gmail):

```
SMTP server: smtp.gmail.com
SMTP server port: 587
Connection security: STARTTLS
Authentication: Username + Password
Username: your-email@gmail.com
Password: your-app-password
```

Example for Office365:

```
SMTP server: smtp.office365.com
SMTP server port: 587
Connection security: STARTTLS
Authentication: Username + Password
Username: your-email@domain.com
Password: your-password
```

Example for local SMTP relay:

```
SMTP server: 127.0.0.1
SMTP server port: 25
Connection security: None
Authentication: None
```

### Save.

---

# ⭐ 3. Assign Email to a User (Admin or your user)

### Go to:
**Administration → Users**

Click your user (e.g., **Admin**).

### Go to the **Media** tab → Add

Fill:

- **Type:** Email  
- **Send to:** your email address  
- **When active:** 1-7, 00:00–24:00  
- **Severity:** Select all (or choose what you want)

Click **Add** → **Update**.

Now Zabbix knows **who** to send alerts to.

---

# ⭐ 4. Enable Actions (Alert Rules)

Zabbix sends alerts only when an **Action** is enabled.

### Go to:
**Alerts → Actions**

Find:

```
Report problems to Zabbix administrators
```

Click it → **Enable**.

This action sends alerts for:

- Host down  
- Service down  
- High CPU  
- Disk full  
- Any trigger

If you want custom alerts (e.g., only SSH or HTTP), you can create your own action — but the default one is enough for now.

---

# ⭐ 5. Test Email Notification

Let’s test if email works.

### Go to:
**Administration → Media types → Email**

Click **Test**.

Enter your email → Click **Test**.

If everything is correct, you will receive:

```
Zabbix test message
```

If not, Zabbix will show the error (SMTP auth, port, TLS, etc.).

---

# ⭐ 6. Trigger a Real Alert (Optional Test)

To test real alerts:

### Option A — Stop Zabbix agent on the client

On the client:

```bash
systemctl stop zabbix-agent
```

Within 60 seconds, you should receive:

```
PROBLEM: Zabbix agent on Centos-Client is unreachable
```

Then start it again:

```bash
systemctl start zabbix-agent
```

You will receive:

```
OK: Zabbix agent on Centos-Client is reachable again
```

### Option B — Stop SSH or HTTP service

SSH:

```bash
systemctl stop sshd
```

HTTP:

```bash
systemctl stop httpd
```

You will get alerts if you attached the templates.

---

# ⭐ Summary (Everything You Need)

| Step | Purpose |
|------|---------|
| Open Zabbix UI | Access dashboard |
| Configure SMTP | Allow Zabbix to send emails |
| Add email to user | Define who receives alerts |
| Enable actions | Turn on alert rules |
| Test email | Verify SMTP works |
| Trigger alert | Confirm notifications |

---



Just tell me what you prefer.
