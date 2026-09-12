# Defender-Wazuh: Active XDR & Incident Response Lab

A complete, high-level security operations center (SOC) simulation environment designed to demonstrate practical endpoint visibility, detection engineering, threat hunting, and automated incident containment.

This project documents the installation, configuration, and verification of a centralized **Wazuh SIEM/XDR Manager** monitoring a multi-OS enterprise virtual network. It highlights advanced systems administration optimizations, PowerShell-driven EDR agent telemetry ingestion, File Integrity Monitoring (FIM) integrated with VirusTotal threat intelligence APIs, and active-defense automation utilizing local host-based firewalls to contain SSH brute-force attacks in real-time.

---

## 🏗️ Lab Architecture & Network Topology

This deployment is configured inside a virtualized hypervisor (VMware Workstation Pro) on an isolated, non-routing **Host-Only Network Segment (`VMnet1`)** to ensure safe, contained execution of offensive security emulations ``.

```
                    +---------------------------------------+
                    |             KALI ATTACKER             |
                    |             192.168.56.20             |
                    +-------------------+-------------------+
                                        |
                                        | (Host-Only Network VMnet1)
                                        |
+---------------------------------------+---------------------------------------+
|                                       |                                       |
|                                       |                                       |
|  +---------------------------------+  |  +---------------------------------+  |  +---------------------------------+
|  |       WAZUH SIEM MANAGER        |  |  |          LINUX VICTIM           |  |  |         WINDOWS VICTIM          |
|  |     Ubuntu 22.04 LTS (OVA)      |  |  |       Ubuntu Server (CLI)       |  |  |       Windows Server Core       |
|  |          192.168.56.2           |  |  |          192.168.56.10          |  |  |          192.168.56.12          |
|  +---------------------------------+  |  +---------------------------------+  |  +---------------------------------+
|  - All-in-One Indexer & Dashboard  |  |  - Target: SSH Brute-Force         |  |  - Target: EDR Logs Ingestion      |
|  - Active Response Coordinator     |  |  - Real-time FIM Directory         |  |  - Sysmon Process Auditing         |
|  - VirusTotal API Threat Analyzer  |  |  - Auto-Firewall (iptables) Block  |  |  - Local GPO Client Constraints    |
+---------------------------------------+---------------------------------------+------------------------------------+

```

### 📋 Node Specifications

* **SIEM Manager:** Wazuh All-in-One Virtual Appliance (OVA) running OpenSearch Indexer, Wazuh Manager, and Kibana-based Dashboard (Allocated: 4 vCPUs, 8GB RAM, 50GB SSD).
* **Linux Victim Endpoint:** Ubuntu Server 22.04 LTS minimal CLI running `openssh-server` and Wazuh Agent.
* **Windows Victim Endpoint:** Windows Server Core (Evaluation) running Wazuh Agent and Microsoft System Monitor (Sysmon).
* **Attacker Node:** Kali Linux running Hydra brute-force simulation and network discovery tools.

---

## 🛠️ Step-by-Step Implementation & Optimization

### Phase 1: Hardened Manager Deployment

To bypass standard `systemd` timeouts and DNS latency hangs on isolated networks, the manager configuration is modified at the system layer before initialization.

1. **Configure Host Hostname Resolution:** Loopback lookups are explicitly defined to prevent internal indexer queries from hanging on unreachable external DNS servers:
```bash
echo "127.0.0.1 $(hostname)" | sudo tee -a /etc/hosts
sudo sed -i 's/^#\?MulticastDNS=.*/MulticastDNS=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved

```


2. **Raise systemd Service Startup Timeouts:** Raised service start boundaries to 600 seconds to ensure the indexing cluster has ample time to settle under resources desync:
```bash
sudo mkdir -p /etc/systemd/system/wazuh-indexer.service.d /etc/systemd/system/wazuh-manager.service.d
sudo tee /etc/systemd/system/wazuh-indexer.service.d/override.conf > /dev/null << 'EOF'

TimeoutStartSec=600
EOF
sudo tee /etc/systemd/system/wazuh-manager.service.d/override.conf > /dev/null << 'EOF'

TimeoutStartSec=600
EOF
sudo systemctl daemon-reload

```
*Note: this change was made to accommodate for the slow startup of the `wazuh-indexer` on my PC. If you are replicating this project on your PC, try with the default configurations first.*


3. **Start and Secure the Indexing Cluster:** Set up a secure custom password with specific character sets to initialize the dashboard:
```bash
sudo systemctl start wazuh-indexer
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p 'YourLabPassword2026.'

```


4. **Confirm Cluster Health:**
```bash
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
sudo /var/ossec/bin/wazuh-control status

```



`!(./screenshots/01_manager_healthy.png)`

---

### Phase 2: Multi-OS Agent Onboarding & Sysmon Ingestion

We deploy lightweight endpoint agents to feed our central analyzer ``. Because the Windows victim runs on a GUI-less Server Core, its EDR configuration is executed entirely via PowerShell.

#### A. Linux Agent Deployment

```bash
# On the Linux Agent, execute the official debian deployment script
wget https://packages.wazuh.com/4.x/wazuh-agent.deb
sudo WAZUH_MANAGER='192.168.56.2' dpkg -i wazuh-agent.deb
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent

```

#### B. Windows Server Core Sysmon EDR Integration (PowerShell-Only)

1. **Register the Windows Agent:** Run the dashboard's deployment string and verify the local service is running:
```powershell
Get-Service WazuhSvc
```


2. **Automated Sysmon & Config Acquisition:** Download Sysmon and Olaf Hartong’s modular configurations ``:
```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Sysmon.zip"
Expand-Archive -Path "C:\Sysmon.zip" -DestinationPath "C:\Sysmon"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/olafhartong/sysmon-modular/master/sysmonconfig.xml" -OutFile "C:\Sysmon\sysmonconfig.xml"

```


3. **Install Sysmon with Config:**
```powershell
cd C:\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```
4.  Inject Sysmon EventChannel into Wazuh Config: Update the local agent's `ossec.conf` configuration dynamically to read the Sysmon event logs:
```powershell
$ossecConf = "C:\Program Files (x86)\ossec-agent\ossec.conf"
$insertText = @"

Microsoft-Windows-Sysmon/Operational
<log_format>eventchannel</log_format>

"@
(Get-Content $ossecConf -Raw) -replace '</ossec_config>', "$insertText`r`n</ossec_config>" | Set-Content $ossecConf
Restart-Service WazuhSvc
```

`!(./screenshots/02_agents_active_sysmon.png)`

---

### Phase 3: FIM & VirusTotal Malware Threat Intelligence

File Integrity Monitoring (FIM) allows us to track unauthorized file system modifications. We pair this with the VirusTotal API on the Wazuh Manager to automatically query file hashes and detect malicious binaries.

1. **Enable Real-time FIM on Agent:** On the **Linux Agent**, edit `/var/ossec/etc/ossec.conf` to monitor the critical downloads path in real-time:
```xml
<directories realtime="yes">/root</directories>
```


Apply change: `sudo systemctl restart wazuh-agent`.
2. **Configure VirusTotal Integration on Manager:** Add the integration block to the manager's `/var/ossec/etc/ossec.conf` file:
```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_VT_API_KEY_HERE</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```


Apply change: `sudo systemctl restart wazuh-manager`.

`!(./screenshots/03_fim_virustotal_alert.png)`

---

### Phase 4: Automated Active Defense (Intrusion Containment)

To transition our SOC lab from passive logging to active containment, we configure **Active Response** ``. When brute-force authentication alerts (Rule 5763) trigger on our Linux Victim, the manager automatically instructs the agent's local host-based firewall to drop the attacker's IP in real-time.

1. **Configure Active Response Rules on Manager:** Edit the manager’s `/var/ossec/etc/ossec.conf` file to bind the local `firewall-drop` script with SSH brute-force triggers:
```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>180</timeout>
</active-response>
```


2. **Simulate SSH Brute-Force from Kali:** Run a Hydra-driven multi-threaded brute-force attack targeting the Linux Victim:
```bash
hydra -l admin -P passwords.txt ssh://192.168.56.10 -t 4
```



`!(./screenshots/04_active_response_triggered.png)`

---

## 🔎 Verification & Forensic Validation

We validate our active defense by verifying that the local host firewall on the Linux Victim has successfully updated its rule chains to isolate the attacker:

1. **Check Local Agent `iptables` Chains:**
```bash
sudo iptables -L -n
```


*Result:* Verify that a `DROP` rule targeting the Kali Attacker's IP (`192.168.56.20`) exists under active chains.
2. **Verify Active Response Logs:**
```bash
cat /var/ossec/logs/active-responses.log
```


*Result:* Confirm that the `firewall-drop` script was successfully invoked with the correct argument (`add` action for the attacker's IP).

`!(./screenshots/05_firewall_validation.png)`

---

## 💡 Key Architectural Insights

* **Least-Noise Telemetry (Sysmon):** By utilizing Olaf Hartong's modular configuration file, our Windows agent only forwards high-fidelity telemetry, ensuring we minimize storage costs and avoid overloading our central indexing database ``.
* **Closed-Loop Incident Containment:** By deploying Wazuh Active Response directly at the local agent layer (`location: local`), we isolate compromised networks at the local firewall boundary, preventing lateral movement before senior analysts need to initiate manual triage ``.
* **Air-Gapped Trust Boundaries:** By optimizing local hostname lookups and configuring local systemd time boundaries, the security operations platform remains fully operational without relying on public DNS resolving paths, adhering to high-security compliance standards.
