**Add Custome Script**:

- In Linux system at location: `vi /var/prtg/scripts/test.sh`
```bash
#!/bin/bash

SERVICE="httpd"

if systemctl is-active --quiet "$SERVICE"; then
    echo "0:100:Service httpd is running"
    exit 0
else
    echo "2:0:Service httpd is NOT running"
    exit 1
fi
```
- Go to PRTG -->Devices-->Add Sensor --> ssh --> SSH Script --> Choose Script -->test.sh -->create.
