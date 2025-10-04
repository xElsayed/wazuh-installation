# 🛡️ Wazuh Installation & Malicious IP Blocking (Hands-on Guide)

Hey everyone 👋  
This repo walks you through installing **Wazuh**, setting up agents, and using **Active Response** to block malicious IPs automatically.  
It’s beginner-friendly, simple, and straight to the point 🚀

---

## 🚀 Quickstart

**Wazuh** is an open-source **XDR and SIEM platform** that provides threat detection, incident response, and monitoring for endpoints and cloud workloads.

In this setup, we’ll install the **Wazuh server**, **indexer**, and **dashboard** all on one host using the installation assistant.

---

## ⚙️ Requirements

| Agents  | CPU     | RAM   | Storage (90 days) |
|----------|----------|-------|------------------|
| 1–25     | 4 vCPU  | 8 GiB | 50 GB            |
| 25–50    | 8 vCPU  | 8 GiB | 100 GB           |
| 50–100   | 8 vCPU  | 8 GiB | 200 GB           |

> 💡 For large deployments, use a distributed setup for better performance and load balancing.

---

## 🖥️ Supported Operating Systems

- Amazon Linux 2 / 2023  
- CentOS Stream 10  
- RHEL 7–10  
- Ubuntu 16.04 – 24.04  

---

## 🧩 Installing Wazuh

Run the following command to install all Wazuh components:

```bash
curl -sO https://packages.wazuh.com/4.13/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
After installation, you’ll see something like this:

pgsql
Copy code
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP>
User: admin
Password: <ADMIN_PASSWORD>
INFO: Installation finished.
Then visit:

cpp
Copy code
https://<WAZUH_DASHBOARD_IP>
🔐 Access Credentials
makefile
Copy code
Username: admin
Password: <ADMIN_PASSWORD>
⚠️ The browser may warn that the SSL certificate isn’t trusted — this is normal for self-signed certificates.

You can view all generated passwords using:

bash
Copy code
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
🚫 Disable Automatic Updates (Recommended)
Prevent accidental upgrades that might break your setup:

bash
Copy code
sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
apt update
🧠 Use Case: Blocking a Malicious Actor
We’ll simulate an attack where a RHEL endpoint tries to access our Ubuntu and Windows web servers.
Wazuh will detect and block the attacker’s IP automatically for 60 seconds using Active Response 🔥

🧱 Infrastructure Setup
Endpoint	Role	Description
RHEL 9	Attacker	Connects to victim web servers
Ubuntu 22.04	Victim	Apache web server + Wazuh Agent
Windows 11	Victim	Apache web server + Wazuh Agent

🐧 Ubuntu Endpoint Setup
Install Apache:

bash
Copy code
sudo apt update
sudo apt install apache2
Allow traffic (if firewall enabled):

bash
Copy code
sudo ufw allow 'Apache'
Check Apache status:

bash
Copy code
sudo systemctl status apache2
Test page:

bash
Copy code
curl http://<UBUNTU_IP>
Configure Wazuh Agent:
Add this inside /var/ossec/etc/ossec.conf:

xml
Copy code
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/apache2/access.log</location>
</localfile>
Restart the agent:

bash
Copy code
sudo systemctl restart wazuh-agent
🪟 Windows Endpoint Setup
Install Apache:

Install the latest Visual C++ Redistributable

Download Apache Win64 ZIP build

Extract Apache24 to C:\

Run:

powershell
Copy code
cd C:\Apache24\bin
.\httpd.exe
Allow access when prompted by Windows Firewall.

Test page:
Open http://<WINDOWS_IP> in your browser.

Configure Wazuh Agent:
Edit C:\Program Files (x86)\ossec-agent\ossec.conf:

xml
Copy code
<localfile>
  <log_format>syslog</log_format>
  <location>C:\Apache24\logs\access.log</location>
</localfile>
Restart Agent:

powershell
Copy code
Restart-Service -Name wazuh
🧠 Wazuh Server Configuration
Download IP Reputation List:

bash
Copy code
sudo yum install -y wget
sudo wget https://iplists.firehol.org/files/alienvault_reputation.ipset -O /var/ossec/etc/lists/alienvault_reputation.ipset
Add the attacker IP:

bash
Copy code
echo "<ATTACKER_IP>" | sudo tee -a /var/ossec/etc/lists/alienvault_reputation.ipset
Convert to CDB format:

bash
Copy code
sudo wget https://wazuh.com/resources/iplist-to-cdblist.py -O /tmp/iplist-to-cdblist.py
sudo /var/ossec/framework/python/bin/python3 /tmp/iplist-to-cdblist.py \
/var/ossec/etc/lists/alienvault_reputation.ipset \
/var/ossec/etc/lists/blacklist-alienvault
Clean up:

bash
Copy code
sudo rm -rf /var/ossec/etc/lists/alienvault_reputation.ipset
sudo rm -rf /tmp/iplist-to-cdblist.py
sudo chown wazuh:wazuh /var/ossec/etc/lists/blacklist-alienvault
⚙️ Add Custom Rule
Edit /var/ossec/etc/rules/local_rules.xml:

xml
Copy code
<group name="attack,">
  <rule id="100100" level="10">
    <if_group>web|attack|attacks</if_group>
    <list field="srcip" lookup="address_match_key">etc/lists/blacklist-alienvault</list>
    <description>IP address found in AlienVault reputation database.</description>
  </rule>
</group>
🧩 Configure Active Response
For Ubuntu Endpoint:
xml
Copy code
<ossec_config>
  <active-response>
    <disabled>no</disabled>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>100100</rules_id>
    <timeout>60</timeout>
  </active-response>
</ossec_config>
For Windows Endpoint:
xml
Copy code
<ossec_config>
  <active-response>
    <disabled>no</disabled>
    <command>netsh</command>
    <location>local</location>
    <rules_id>100100</rules_id>
    <timeout>60</timeout>
  </active-response>
</ossec_config>
Restart Wazuh Manager:

bash
Copy code
sudo systemctl restart wazuh-manager
⚔️ Attack Simulation
From the attacker endpoint (RHEL):

bash
Copy code
curl http://<WEBSERVER_IP>
✅ First request goes through.
🚫 Next ones get blocked for 60 seconds.

📊 View Alerts
Open Wazuh Dashboard → Threat Hunting
Search using this query:

python
Copy code
Ubuntu - rule.id:(651 OR 100100)
You’ll see real-time alerts and responses 💥

🧹 Uninstall Wazuh (Optional)
bash
Copy code
sudo bash ./wazuh-install.sh -u
💬 Final Notes
Wazuh is free & open source 🧡

Play around with Active Response — it’s super powerful.

Always snapshot or back up your environment before major changes.

