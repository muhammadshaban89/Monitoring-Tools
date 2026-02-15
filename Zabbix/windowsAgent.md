How to Install Zabbix Agent on Windows 
---------------------------------------

1. Download Windows Agent
From official Zabbix site:
https://www.zabbix.com/download_agents


2. Install the agent
Run:
zabbix_agentd.exe --install
zabbix_agentd.exe --start


3. Configure agent
Edit:
C:\Program Files\Zabbix Agent\zabbix_agentd.conf


Set:
Server=YOUR_ZABBIX_SERVER_IP
ServerActive=YOUR_ZABBIX_SERVER_IP
Hostname=WINDOWS-HOSTNAME


4. Allow firewall
netsh advfirewall firewall add rule name="Zabbix Agent" dir=in action=allow protocol=TCP localport=10050


5. Add host in Zabbix UI
Use template:
Template OS Windows by Zabbix agent


==---
