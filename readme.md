# SOC-in-a-Box: Proxmox Virtual Lab Setup

![](Attachments/SOC-in-a-box%20topology.drawio.svg)

## Stack List

### Hypervisor & Infrastructure

* **Hypervisor**: Proxmox VE

* **Network Security**: OPNsense 26.7

### Detection & Response

* **XDR/EDR**: Wazuh 4.14 running on Ubuntu Server 24.04 LTS

* **SOAR**: Shuffle running on Ubuntu Server 24.04 LTS (Separate from the Wazuh server)

* **Case Management**: TheHive running on Ubuntu Server 24.04 LTS (Same server as Shuffle)

### Targets

* **Windows Ecosystem**: Windows Server 2022 (Domain Controller) and Windows 11 (Workstation(s))

* **Linux Ecosystem**: Ubuntu Server 24.04 LTS (Web/Database) running a vulnerable LEMP stack

* **Attacker Toolset**: Kali Linux (Metasploit, Nmap, Burp Suite)

## Deployment Guide

### Phase 1: Hypervisor & Networking Setup

To ensure realistic SOC monitoring and attacker/victim isolation, we utilize VLAN segmentation across an isolated bridge (`vmbr2`) handled by an OPNsense virtual router/firewall.

**Architecture Overview:**

* **Internal Bridge:** `vmbr2` (Isolated, VLAN-aware, no physical bridge ports)

* **Firewall / Gateway:** OPNsense VM (WAN on `vmbr0`, LAN Trunk on `vmbr2`)

* **VLAN 20 (Servers):** `10.0.20.0/24` (Gateway: `10.0.20.1`) - *Domain Controller, Web Server, SOC Automation*

* **VLAN 30 (Clients / Workstations):** `10.0.30.0/24` (Gateway: `10.0.30.1`) - *Windows 11 targets*

* **VLAN 40 (Tools / Attackers):** `10.0.40.0/24` (Gateway: `10.0.40.1`) - *Kali Linux, testing tools*

#### 1. Proxmox Hypervisor Setup

1. In the Proxmox Web GUI, navigate to **System > Network**.

2. Create a new **Linux Bridge**:

   * **Name:** `vmbr2`

   * **IPv4 / Gateway:** Leave blank

   * **VLAN aware:** Checked (`Yes`)

   * **VLAN IDs:** `2-4094` (default)

   * **Bridge ports:** Leave completely blank to keep traffic isolated from the host physical network.

3. Click **Apply Configuration**.

#### 2. OPNsense VM Configuration

1. Deploy the OPNsense VM with 2 cores, 4GB memory, and 32GB storage.

2. Ensure the LAN Network Device is attached to **Bridge:** `vmbr2` using the **VMware vmxnet3** or **VirtIO** model. Uncheck the firewall box. Under Advanced, set **Trunks:** to `2-4094`.

3. Inside the OPNsense GUI, go to **Interfaces > Other Types > VLAN** and add:

   * **VLAN 20:** Parent: `vmx0` (LAN) | Tag: `20` | Description: `VLAN20_Servers`

   * **VLAN 30:** Parent: `vmx0` (LAN) | Tag: `30` | Description: `VLAN30_Clients`

   * **VLAN 40:** Parent: `vmx0` (LAN) | Tag: `40` | Description: `VLAN40_Attackers`

4. Go to **Interfaces > Assignments**, assign them (e.g., `OPT1`, `OPT2`, `OPT3`), and configure Static IPv4 addresses (`10.0.20.1/24`, `10.0.30.1/24`, and `10.0.40.1/24`).

5. Go to **Firewall > Rules** for each VLAN and add a temporary "Pass All" rule (Action: Pass, Direction: in, Protocol: Any, Source/Dest: Any) to establish connectivity.

#### 3. Kea DHCP & Unbound DNS

1. Disable legacy services: Go to **Services > DHCRelay** and **Services > Dnsmasq DNS & DHCP** and ensure both are unchecked/disabled to free up port 67.

2. Enable Kea DHCP: Go to **Services > KEA DHCP > KEA DHCPv4 > Settings**. Enable the server and select `LAN`, `VLAN 20`, `VLAN 30`, and `VLAN 40` as listening interfaces.

3. Configure Kea Subnets for `10.0.20.0/24`, `10.0.30.0/24`, and `10.0.40.0/24`, setting the respective gateway IP as the DNS Server for each. Restart Kea.

4. Enable Unbound DNS: Go to **Services > Unbound DNS > General**. Enable it and set Network Interfaces to **All**.

5. Go to **Unbound DNS > Access Lists** and add explicit `Allow` rules for the `10.0.20.0/24`, `10.0.30.0/24`, and `10.0.40.0/24` networks. Restart Unbound.

### Phase 2: Windows Target Deployment

*Note: Windows requires specific drivers to function on Proxmox KVM.*
#### 1. VM Provisiong & OS Installation

1. **VM Creation:** Create a VM using `OVMF (UEFI)` and `q35` machine type. Assign 4 cores, 8GB memory, and 64GB storage.

2. **Network Assignment (Windows 11 Workstation):** Assign the network device to Bridge `vmbr2` and set the **VLAN Tag** to `30`. Uncheck the Proxmox firewall box.

3. **Driver Injection:** During the Windows installation, attach the **VirtIO Win ISO**.

4. **Driver Loading:** Use `Load Driver` to manually point to the `viostor` (disk) and `NetKVM` (network) directories to ensure high-performance hardware abstraction.

5. **Hardening:** Implement **TPM 2.0 (vTPM)** via Proxmox to enable Windows Credential Guard and BitLocker.

6. **Windows Server 2022 (Domain Controller):** Repeat these steps for the Windows Server, but assign the network adapter **VLAN Tag `20`**. Assign 2 cores, 4GB memory, and 40GB storage. Make sure during installation to select one that does not say "with GUI" or you will need to allot more cores and memory.
#### 2. Domain Controller Setup (Windows Server 2022 Core)

Boot the Windows Server VM and drop into the local PowerShell console.

**Set Static IP & Hostname:**
```powershell
# Identify your network adapter name (usually "Ethernet")
Get-NetAdapter | Format-Table Name, Status, MacAddress

# Set a static IP on VLAN 20
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.20.169 -PrefixLength 24 -DefaultGateway 10.0.20.1

# Temporarily point DNS to OPNsense/External
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 8.8.8.8

# Rename the server and restart
Rename-Computer -NewName "DC01" -Restart
```

**Time Synchronization (Crucial):**
*Kerberos authentication will reject clients if the DC and client clocks are more than 5 minutes out of sync.*
```powershell
# Set correct timezone
Set-TimeZone -Id "Eastern Standard Time"

# Sync with NTP
w32tm /config /manualpeerlist:"0.pool.ntp.org,0x1 1.pool.ntp.org,0x1" /syncfromflags:manual /reliable:YES /update
net stop w32time
net start w32time
w32tm /resync /force
```

**Install AD DS & Promote:**
```powershell
Install-WindowsFeature ADDS-Domain-Controller -IncludeManagementTools

# Promote to DC (This will interactively prompt for a DSRM password)
Install-ADDSForest -DomainName "corp.lab" -InstallDns:$true -DomainNetBIOSName "CORP"
```
*The server will reboot automatically.*

**Post-Promotion Housekeeping:**
```powershell
# Point DNS to itself
netsh interface ip set dns name="Ethernet" static 10.0.20.169

# Setup DNS Forwarders to OPNsense
Set-DnsServerForwarder -IPAddress 10.0.20.1, 8.8.8.8

# Create an Admin OU and user
New-ADOrganizationalUnit -Name "Admins"
$pass = Read-Host "Enter Password" | ConvertTo-SecureString -AsPlainText -Force
New-ADUser -Name "labadmin" -SamAccountName "labadmin" -UserPrincipalName "labadmin@corp.lab" -Path "OU=Admins,DC=corp,DC=lab" -AccountPassword $pass -Enabled $true

# Add user to Domain Admins
Add-ADGroupMember "Domain Admins" -Members "labadmin"
```

**Disable Cross-Subnet Firewall Blocking (Crucial Lab Step):**
If you are having issues with step 3 try temporarily disabling the firewall, or adjust the AD rule scopes to allow VLAN 30.
```powershell
# Temporarily disable the firewall to allow inter-VLAN domain joins
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

#### 3. Client Domain Join (Windows 11)

Before attempting to join the domain from your VLAN 30 Windows 11 machine, verify these prerequisites:
1. **Edition:** You **must** be running Windows 11 Pro, Education, or Enterprise. Windows 11 Home does not support AD domains. *(Lab Tip: Upgrade offline using the generic key `VK7JG-NPHTM-C97JM-9MPGT-3V66T`).*
2. **Time Sync:** Ensure the Windows 11 clock matches the DC exactly.
3. **DNS:** Point the Windows 11 IPv4 DNS directly to the DC (`10.0.20.169`).

**Execute the Join:**
1. Open the Start Menu, search for **PowerShell**, right-click, and select **Run as Administrator**. *(Running standard PowerShell will result in a local "Access is denied" error).*
2. Run the join command:
```powershell
Add-Computer -DomainName "corp.lab" -Credential corp.lab\Administrator -Restart
```
Once joined and rebooted, verify the secure channel by running `nltest /sc_query:corp.lab`. Don't forget to go back to the DC and re-enable the firewall if you disabled it (`Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True`).
### Phase 3: Linux Web Server Deployment (VLAN 20)

Deploy an Ubuntu Server instance with 1 core, 1GB memory, and 32GB storage. Assign the network adapter to Bridge `vmbr2` with a **VLAN Tag** of `20`.

#### 1. OS & Network Setup

1. Deploy an Ubuntu 24.04 instance.

2. Configure a static IP using `netplan` or reserve an IP in OPNsense (e.g., `10.0.20.100/24`).

3. Configure the firewall (`ufw`) to allow essential traffic:

```
sudo ufw allow 22/tcp # SSH
sudo ufw allow 80/tcp # HTTP
sudo ufw allow 443/tcp # HTTPS
sudo ufw enable
```

#### 2. LEMP Stack Installation

```
sudo apt update && sudo apt install nginx mariadb-server php-fpm php-mysql -y

```

#### 3. DVWA (Damn Vulnerable Web App) Deployment

This provides a controlled attack surface for testing SQLi and XSS payloads.

**Download & Extract:**

```
cd /var/www/html
sudo wget https://github.com/digininja/DVWA/archive/master.zip
sudo apt install unzip -y
sudo unzip master.zip
```

**Set Permissions:**

```
sudo chown -R www-data:www-data /var/www/html/DVWA-master
sudo chmod -R 755 /var/www/html/DVWA-master
sudo chmod -R 777 /var/www/html/DVWA-master/config
```

**Database Configuration:**

```
sudo mysql -u root
CREATE DATABASE dvwa;
CREATE USER 'dvwa_user'@'localhost' IDENTIFIED BY 'password123';
FLUSH PRIVILEGES;
EXIT;
```

**Finalize App Config:**

```
cd /var/www/html/DVWA-master/config
sudo cp config.inc.php.dist config.inc.php
sudo nano config.inc.php # Set db_user and db_password to match the database setup
```

**DVWA is now available at http://<your_VM_IP>/DVWA-master**

### Phase 4: SOC Automation Stack (VLAN 20)

To maintain environment stability, the automation layer is deployed via **Docker Compose** on a dedicated Ubuntu Automation Server. Assign these VMs to Bridge `vmbr2` with a **VLAN Tag** of `20`. 

*   **Automation Server (Shuffle/TheHive):** 2 cores, 12GB memory, 100GB storage.
*   **Wazuh Server:** 4 cores, 8GB memory, 64GB storage.

#### 1. System Preparation & Docker Setup

Update the host and install the Docker engine alongside Git:

```
sudo apt update && sudo apt install docker.io docker-compose-v2 git -y
```

Grant your user account permissions to run Docker commands without root:

```
sudo usermod -aG docker $USER
newgrp docker
```

Ensure Docker service is set to start automatically when the VM boots:

```
sudo systemctl enable docker
```

#### 2. Deploying Shuffle (SOAR)

Clone the official repository and spin up the containers:

```
git clone https://github.com/Shuffle/Shuffle.git
cd Shuffle
docker compose up -d
```

**Shuffle will be accessible at https://<your_VM_IP>:3443**

#### 3. Deploying TheHive (Case Management)

Clone the official repository:

```
git clone https://github.com/StrangeBeeCorp/docker.git
```

Navigate to selected profile:

```
cd docker/prod1-thehive
```

Initialize the environment using the provided init script:

```
bash ./scripts/init.sh
```

Limit JVM heaps in `docker-compose.yml` using nano:

* Under the `elasticsearch` environment section, edit/add: `- ES_JAVA_OPTS=-Xms1g -Xmx1g`

* Under the `cassandra` environment section, add: `- MAX_HEAP_SIZE=1G` and `- HEAP_NEWSIZE=1G`

Start TheHive:

```
docker compose up -d
```

**TheHive will be accessible at http://<your_VM_IP>:9000**

#### 4. Container Persistence

Ensure the SOC automation pipeline survives reboots by applying `unless-stopped`:

```
docker update --restart unless-stopped $(docker ps -q)
```

### Phase 5: Log Aggregation & Telemetry

To provide complete visibility for the SOC, we deploy endpoint agents to the "Target" machines and route perimeter network telemetry (syslog) from the firewall into the SIEM.

#### 1. Wazuh Agent Deployment Strategy
Install the Wazuh Agent software *only* on the intended target machines to observe endpoint events (e.g., lateral movement, malware execution). 
* **Target 1:** Windows 11 Workstation
* **Target 2:** Windows Server 2022 (Domain Controller)
* **Target 3:** Ubuntu Web Server (DVWA)

*(Note: Do not install the agent on the Kali attack box or the OPNsense firewall).*

#### 2. Enable Wazuh Syslog Listener
To ingest firewall logs, the Wazuh Manager must be configured to listen for UDP syslog traffic.
1. On the Ubuntu Wazuh Server, edit `/var/ossec/etc/ossec.conf`.
2. Add a new `<remote>` block (do not place this inside the existing secure 1514 block):
```xml
  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>YOUR_OPNSENSE_IP</allowed-ips>
  </remote>
```
3. Run `sudo /var/ossec/bin/wazuh-control checkconf` to verify XML syntax.
4. Restart the manager: `sudo systemctl restart wazuh-manager`.
5. Verify the listener is active: `sudo ss -uln | grep 514`.

#### 3. Enable Wazuh Archiving (Raw Logs)
Because raw firewall logs do not trigger alerts by default, you must enable archiving to view them in the dashboard.
1. Enable JSON logging in `/var/ossec/etc/ossec.conf` by changing `<logall_json>no</logall_json>` to `<logall_json>yes</logall_json>`. Restart the Wazuh Manager.
2. Tell Filebeat to forward the archives by editing `/etc/filebeat/filebeat.yml`. Change `archives: enabled: false` to `true`. Restart the Filebeat service (`sudo systemctl restart filebeat`).
3. In the Wazuh Dashboard, go to **Dashboard Management > Index patterns**, create an index named `wazuh-archives-*`, and select `timestamp`. You can now view raw logs in the **Discover** tab.

#### 4. Configure OPNsense Log Forwarding
1. In the OPNsense GUI, go to **System > Settings > Logging**.
2. Add a new destination: **UDP**, Port **514**, Hostname **<WAZUH_SERVER_IP>**.
3. Select desired applications (e.g., `filterlog`, `suricata`) and apply changes.
4. *(Optional Testing)*: To generate noise, ensure logging is enabled on your specific firewall rules (click the 'i' icon next to the rule to turn it blue) and run a noisy Nmap scan (`nmap -sS -p- -T4`) from Kali. 

### Phase 6: Automated Alert Pipeline (Wazuh to TheHive)

This phase connects the detection engine (Wazuh) to the Case Management platform (TheHive) using the SOAR tool (Shuffle) to automatically generate SOC alerts.

#### 1. Wazuh Manager Configuration
Define the Shuffle integration in the Wazuh Manager configuration file so it knows where to send alerts.

1. SSH into the Wazuh Manager.
2. Edit the configuration file: `sudo nano /var/ossec/etc/ossec.conf`
3. Add the following `<integration>` block just under the `global` block(adjust `<level>` to filter noise, and update the `hook_url` with your Shuffle webhook):
   ```xml
   <integration>
       <name>shuffle</name>
       <hook_url>https://<SHUFFLE_IP>:3443/api/v1/hooks/webhook_<ID></hook_url>
       <level>3</level>
       <alert_format>json</alert_format>
   </integration>
   ```

#### 2. Bypass Self-Signed SSL Certificates
If Shuffle is using a self-signed certificate (standard in local lab setups), Wazuh's default integration script will silently drop the payload due to a certificate verification failure. 

1. Open the native Shuffle integration Python script:
   `sudo nano /var/ossec/integrations/shuffle.py`
2. Locate the `requests.post()` method near the bottom of the script.
3. Append `, verify=False` to the request parameters. It should look like this:
   ```python
   res = requests.post(url, data=msg, headers=headers, timeout=10, verify=False)
   ```
4. Save the file and restart the Wazuh Manager service to apply all changes:
   `sudo systemctl restart wazuh-manager`

#### 3. Shuffle Workflow & Payload Mapping
With Wazuh pushing data, configure Shuffle to receive the webhook and format the payload for TheHive.

1. Create a new workflow in Shuffle with a **Webhook** trigger.
2. Add a **TheHive 5** app node to the canvas and link the Webhook to it.
3. Configure the TheHive node action to **Create alert**.
4. Switch the configuration panel to the **Advanced** tab to map the dynamic Wazuh data into the raw JSON body using the Execution Argument variable picker (`+`).

**Example TheHive JSON Payload:**
```json
{
  "title": "$exec.text.rule.description",
  "description": "Target: $exec.text.data.dstuser | Source IP: $exec.text.data.srcip \n\nRaw Log: $exec.text.full_log",
  "severity": ${severity},
  "sourceRef": "$exec.id",
  "type": "wazuh",
  "source": "wazuh"
}
```
*Note: The `sourceRef` field must be unique for every alert to bypass TheHive's deduplication filtering. Mapping this to the unique Shuffle execution ID (`$exec.id`) guarantees delivery.*

### Next Steps

1. Write custom Wazuh Decoders and Rules to parse OPNsense `filterlog` traffic into actionable Dashboard Alerts.
2. Expand the SOAR pipeline in Shuffle by extracting IP Observables and routing them through a Threat Intelligence feed (like AbuseIPDB).
3. Generate active response playbooks to automatically isolate compromised endpoints.