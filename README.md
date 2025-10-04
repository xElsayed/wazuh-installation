# wazuh-installation
🛡️ Wazuh Quickstart & Blocking Malicious IPs (Hands-on Guide)

Hey everyone 👋
This repo walks you through installing Wazuh, setting up agents, and using Active Response to block malicious IPs automatically.
It’s beginner-friendly, simple, and straight to the point.

🚀 Quickstart

Wazuh is an open-source XDR and SIEM platform that provides threat detection, incident response, and monitoring for endpoints and cloud workloads.

In this setup, we’ll install the Wazuh server, indexer, and dashboard all on one host using the Wazuh installation assistant.

⚙️ Requirements
Agents	CPU	RAM	Storage (90 days)
1–25	4 vCPU	8 GiB	50 GB
25–50	8 vCPU	8 GiB	100 GB
50–100	8 vCPU	8 GiB	200 GB

💡 For large deployments, use a distributed setup for better performance and load balancing.

🖥️ Supported Operating Systems

Amazon Linux 2 / 2023

CentOS Stream 10

RHEL 7–10

Ubuntu 16.04 – 24.04

🧩 Installing Wazuh

Run the following command to download and install Wazuh’s central components:

curl -sO https://packages.wazuh.com/4.13/wazuh-install.sh && sudo bash ./wazuh-install.sh -a


After installation, you’ll see something like:

INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP>
    User: admin
    Password: <ADMIN_PASSWORD>
INFO: Installation finished.


Now open your browser and visit:

https://<WAZUH_DASHBOARD_IP>

🔐 Access Credentials

Username: admin

Password: <ADMIN_PASSWORD>

⚠️ Your browser might warn that the SSL certificate isn’t trusted — that’s normal for self-signed certs.

You can find all generated passwords here:

sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt

🚫 Disable Automatic Updates (Recommended)

To prevent Wazuh from auto-updating and breaking the setup, disable its repo:

sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
apt update

🧠 Use Case: Blocking a Malicious Actor

We’ll simulate a scenario where an attacker (RHEL endpoint) tries to access our Ubuntu and Windows web servers.
Wazuh will detect the malicious IP and block it automatically for 60 seconds using Active Response.

🧱 Infrastructure Setup
Endpoint	Role	Description
RHEL 9	Attacker	Connects to victim web servers
Ubuntu 22.04	Victim	Apache web server + Wazuh Agent
Windows 11	Victim	Apache web server + Wazuh Agent
🐧 Ubuntu Endpoint Setup

1. Install Apache:

sudo apt update
sudo apt install apache2


2. Allow traffic (if firewall enabled):

sudo ufw allow 'Apache'


3. Verify Apache is running:

sudo systemctl status apache2


4. Check the landing page:

curl http://<UBUNTU_IP>


5. Configure the Wazuh Agent:
Add this to /var/ossec/etc/ossec.conf:

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/apache2/access.log</location>
</localfile>


6. Restart the Wazuh agent:

sudo systemctl restart wazuh-agent

🪟 Windows Endpoint Setup

1. Install Apache:

Install the latest Visual C++ Redistributable

Download Apache Win64 ZIP build

Extract Apache24 to C:\

Run Apache:

cd C:\Apache24\bin
.\httpd.exe


Allow access when prompted by Windows Firewall.

2. Check the landing page:
Open:

http://<WINDOWS_IP>


3. Configure Wazuh Agent:
Edit C:\Program Files (x86)\ossec-agent\ossec.conf:

<localfile>
  <log_format>syslog</log_format>
  <location>C:\Apache24\logs\access.log</location>
</localfile>


4. Restart Agent:

Restart-Service -Name wazuh

🧠 Wazuh Server Configuration
Download IP Reputation List
sudo yum install -y wget
sudo wget https://iplists.firehol.org/files/alienvault_reputation.ipset -O /var/ossec/etc/lists/alienvault_reputation.ipset

Add the Attacker IP
echo "<ATTACKER_IP>" | sudo tee -a /var/ossec/etc/lists/alienvault_reputation.ipset

Convert to CDB format
sudo wget https://wazuh.com/resources/iplist-to-cdblist.py -O /tmp/iplist-to-cdblist.py
sudo /var/ossec/framework/python/bin/python3 /tmp/iplist-to-cdblist.py \
/var/ossec/etc/lists/alienvault_reputation.ipset \
/var/ossec/etc/lists/blacklist-alienvault

Clean up
sudo rm -rf /var/ossec/etc/lists/alienvault_reputation.ipset
sudo rm -rf /tmp/iplist-to-cdblist.py
sudo chown wazuh:wazuh /var/ossec/etc/lists/blacklist-alienvault

⚙️ Add Custom Rule

Edit /var/ossec/etc/rules/local_rules.xml:

<group name="attack,">
  <rule id="100100" level="10">
    <if_group>web|attack|attacks</if_group>
    <list field="srcip" lookup="address_match_key">etc/lists/blacklist-alienvault</list>
    <description>IP address found in AlienVault reputation database.</description>
  </rule>
</group>

🧩 Configure Active Response
For Ubuntu Endpoint:
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
<ossec_config>
  <active-response>
    <disabled>no</disabled>
    <command>netsh</command>
    <location>local</location>
    <rules_id>100100</rules_id>
    <timeout>60</timeout>
  </active-response>
</ossec_config>


Restart the manager:

sudo systemctl restart wazuh-manager

⚔️ Attack Simulation

From the attacker (RHEL endpoint), run:

curl http://<WEBSERVER_IP>


First request goes through.
Next ones get blocked for 60 seconds 🔥

📊 View Alerts

Open the Wazuh Dashboard → Threat Hunting
Use this query:

Ubuntu - rule.id:(651 OR 100100)


You’ll see the alert logs and response actions in real-time.

🧹 Uninstall Wazuh (Optional)

If you ever wanna remove everything:

sudo bash ./wazuh-install.sh -u

💬 Final Notes

Wazuh is free & open source 🧡

Play around with the Active Response module — it’s powerful.

Don’t forget to snapshot or backup your environment before major changes.
