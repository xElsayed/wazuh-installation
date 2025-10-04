# 🛡️ Wazuh Quickstart & Blocking Malicious IPs (Hands-on Guide)

Hey everyone 👋  
This repo walks you through installing **Wazuh**, setting up agents, and using **Active Response** to block malicious IPs automatically.  
It’s beginner-friendly, simple, and straight to the point.

---

## 🚀 Quickstart

**Wazuh** is an open-source XDR and SIEM platform that provides threat detection, incident response, and monitoring for endpoints and cloud workloads.

In this setup, we’ll install the **Wazuh server**, **indexer**, and **dashboard** all on one host using the Wazuh installation assistant.

---

## ⚙️ Requirements

| Agents | CPU | RAM | Storage (90 days) |
|--------:|----:|----:|------------------:|
| 1–25 | 4 vCPU | 8 GiB | 50 GB |
| 25–50 | 8 vCPU | 8 GiB | 100 GB |
| 50–100 | 8 vCPU | 8 GiB | 200 GB |

💡 For large deployments, use a distributed setup for better performance and load balancing.

---

## 🖥️ Supported Operating Systems

- Amazon Linux 2 / 2023  
- CentOS Stream 10  
- RHEL 7–10  
- Ubuntu 16.04 – 24.04

---

## 🧩 Installing Wazuh

Run this command to install Wazuh’s core components:

```bash
curl -sO https://packages.wazuh.com/4.13/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

After installation, you’ll see:

```
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP>
User: admin
Password: <ADMIN_PASSWORD>
INFO: Installation finished.
```

Then visit:
```
https://<WAZUH_DASHBOARD_IP>
```

---

## 🔐 Access Credentials

- **Username:** admin  
- **Password:** `<ADMIN_PASSWORD>`  

⚠️ Your browser might warn that the SSL certificate isn’t trusted — that’s normal for self-signed certs.

To view generated passwords:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

---

## 🚫 Disable Automatic Updates (Recommended)

```bash
sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
apt update
```

---

## 🧠 Use Case: Blocking a Malicious Actor

Simulate an attacker (RHEL endpoint) trying to access Ubuntu & Windows servers.  
Wazuh detects the malicious IP and blocks it automatically for 60 seconds using **Active Response**.

---

## 🧱 Infrastructure Setup

| Endpoint | Role | Description |
|----------|------|-------------|
| RHEL 9 | Attacker | Connects to victim web servers |
| Ubuntu 22.04 | Victim | Apache web server + Wazuh Agent |
| Windows 11 | Victim | Apache web server + Wazuh Agent |

---

### 🐧 Ubuntu Endpoint Setup

```bash
sudo apt update
sudo apt install apache2
sudo ufw allow 'Apache'
sudo systemctl status apache2
curl http://<UBUNTU_IP>
```

Edit `/var/ossec/etc/ossec.conf` and add:
```
syslog /var/log/apache2/access.log
```
Then restart the agent:
```bash
sudo systemctl restart wazuh-agent
```

---

### 🪟 Windows Endpoint Setup

1. Install the latest Visual C++ Redistributable  
2. Download Apache Win64 ZIP build  
3. Extract to `C:\Apache24`  
4. Run:
   ```powershell
   cd C:\Apache24\bin
   .\httpd.exe
   ```
5. Allow access when prompted by Windows Firewall  
6. Visit `http://<WINDOWS_IP>` to verify.  

Edit `C:\Program Files (x86)\ossec-agent\ossec.conf`:
```
syslog C:\Apache24\logs\access.log
```
Then restart the agent:
```powershell
Restart-Service -Name wazuh
```

---

## 🧠 Wazuh Server Configuration

### Download IP Reputation List

```bash
sudo yum install -y wget
sudo wget https://iplists.firehol.org/files/alienvault_reputation.ipset -O /var/ossec/etc/lists/alienvault_reputation.ipset
```

### Add Attacker IP
```bash
echo "<ATTACKER_IP>" | sudo tee -a /var/ossec/etc/lists/alienvault_reputation.ipset
```

### Convert to CDB format
```bash
sudo wget https://wazuh.com/resources/iplist-to-cdblist.py -O /tmp/iplist-to-cdblist.py
sudo /var/ossec/framework/python/bin/python3 /tmp/iplist-to-cdblist.py /var/ossec/etc/lists/alienvault_reputation.ipset /var/ossec/etc/lists/blacklist-alienvault
```

### Clean up
```bash
sudo rm -rf /var/ossec/etc/lists/alienvault_reputation.ipset
sudo rm -rf /tmp/iplist-to-cdblist.py
sudo chown wazuh:wazuh /var/ossec/etc/lists/blacklist-alienvault
```

---

## ⚙️ Add Custom Rule

Edit `/var/ossec/etc/rules/local_rules.xml`:

```xml
<rule id="100100" level="10">
  <if_sid>651</if_sid>
  <list>etc/lists/blacklist-alienvault</list>
  <description>IP found in AlienVault reputation database</description>
  <group>web,attack,attacks</group>
</rule>
```

---

## 🧩 Configure Active Response

**Ubuntu Endpoint:**
```xml
<ossec_config>
  <active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>100100</rules_id>
    <timeout>60</timeout>
  </active-response>
</ossec_config>
```

**Windows Endpoint:**
```xml
<ossec_config>
  <active-response>
    <command>netsh</command>
    <location>local</location>
    <rules_id>100100</rules_id>
    <timeout>60</timeout>
  </active-response>
</ossec_config>
```

Restart the manager:
```bash
sudo systemctl restart wazuh-manager
```

---

## ⚔️ Attack Simulation

From the attacker (RHEL endpoint), run:
```bash
curl http://<WEBSERVER_IP>
```

✅ First request goes through.  
🚫 Next ones get blocked for 60 seconds.

---

## 📊 View Alerts

Open **Wazuh Dashboard → Threat Hunting** and use the query:

```
Ubuntu - rule.id:(651 OR 100100)
```

You’ll see alert logs and response actions in real-time.

---

## 🧹 Uninstall Wazuh (Optional)

```bash
sudo bash ./wazuh-install.sh -u
```

---

## 💬 Final Notes

- Wazuh is free & open source 🧡  
- Play around with the Active Response module — it’s powerful.  
- Don’t forget to **snapshot or backup** your environment before major changes.
