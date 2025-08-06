<img width="1376" height="584" alt="image" src="https://github.com/user-attachments/assets/67ee9cf5-3db5-4ed5-9b0e-0baea2aaa91a" /># Multisite Active Directory Collaborated Lab with IPsec VPN and Security Monitoring
Welcome to the multisite active directory lab with site-to-site VPN configuration and implemented security measures. This lab has been created in collaboration with [edris526](https://github.com/edris526).

![PROJECT-MAP](images/project-diagram.png)

This project simulates a realistic enterprise network environment across three geographically separated Active Directory (AD) sites, interconnected through IPSec VPN tunnels. It provides a platform for blue team monitoring and red team simulation, suitable for learning, research, and testing in cybersecurity and system administration.

# Project Overview

## 🌐 Domain: `project.lab`

### 🏢 Sites Overview

- **KBL SITE** (`192.168.1.0/24`)  
  - Server: `KBL-SRV`  
  - Roles: Splunk Universal Forwarder, Sysmon  
  - VPN Connection: IPSec Tunnel to MZR and HRT


- **HRT SITE** (`192.168.2.0/24`)  
  - Domain Controller: `HRT-SRV (ADC)`
      - Roles: Splunk Universal Forwarder, Sysmon 
  - Ubentu Server: Splunk Indexer  
  - Endpoints:
    - `Windows 10 Client`: Atomic Red Team
    - `Kali Linux`: Simulated attacker system  
  - VPN Connection: IPSec Tunnel to MZR and KBL

- **MZR SITE** (`192.168.3.0/24`)  
  - Server: `MZR-SRV`  
  - Roles: Splunk Universal Forwarder, Sysmon  
  - VPN Connection: IPSec Tunnel to KBL and HRT

---

## 🔐 Security & Monitoring Stack

- **IPSec Site-to-Site VPN**  
  Ensures secure and encrypted communication between the AD sites using virtual tunnel interfaces (VTIs).

- **Splunk Monitoring**
  - **Indexer** hosted at HRT
  - **Universal Forwarders** installed on all servers
  - **Sysmon** for endpoint visibility and Windows telemetry

- **Atomic Red Team (ART)**
  - Deployed at **HRT site** to simulate adversary behavior for security testing and detection engineering.

- **Red Team Simulation**
  - `Kali Linux` machine performs **brute-force attacks** on the Windows 10 system's **RDP service** using the **Hydra tool**.
  - Attack telemetry is collected and analyzed via Splunk.

---

## 🎯 Objectives

- Build a secure, isolated, and production-like multi-site AD infrastructure.
- Implement encrypted routing between sites using IPSec VPN.
- Gain hands-on experience with log aggregation and detection engineering using Splunk.
- Test endpoint detection and response (EDR) strategies with Sysmon and ART.
- Simulate real-world attacks and analyze their artifacts in a monitored environment.

---

## 📁 Tools & Technologies

- **VMware:** used for virtualization of network components (VyOS routers and end-hosts)
- **Windows Server 2019/2022**, **Windows 10**
- **Kali Linux**
- **Splunk Enterprise**, **Splunk Universal Forwarder**
- **Sysmon**
- **Atomic Red Team**
- **Hydra**
- **VyOS**

---

## 🧰 Prerequisites
Before setting up and running this lab environment, ensure you meet the following system and knowledge requirements:
- A host system capable of virtualization (e.g., 16+ GB RAM, 4+ core CPU, 100+ GB disk space)
- **VMware Workstation**, **VMware Pro**, or similar hypervisor software

You should be comfortable with:

- Creating and managing virtual machines in **VMware**
- Installing and configuring operating systems (Windows Server, Windows 10, Kali Linux)
- Importing **VyOS** router images and setting up virtual networking (NAT, Host-Only, or Bridged)

To successfully deploy and configure the environment, you should have **basic to intermediate knowledge** of:

- **Active Directory Domain Services (AD DS)**  
  - Domain controller roles and promotion process  
  - DNS and domain name configuration  
  - AD Sites and Services for multi-site replication

- **Networking Basics**  
  - IP addressing, subnets, and routing  
  - Basic VPN/IPSec concepts  
  - DHCP and DNS principles

- **Command Line Tools**  
  - PowerShell (for Windows Server configuration)
  - Linux CLI (for Kali Linux and VyOS management)
  
---

## 🧠 Ideal For:

- Students and professionals learning enterprise AD topology
- Blue teamers practicing log analysis and detection
- Red teamers testing adversary simulation in isolated labs
- SOC analysts refining Splunk dashboards and alerts

---

# Network Configuration

We are going to use VMware for the servers and routers virtualization. In this project, all routers and servers have been previously installed so you need to be able to install virtual machines such as Windows Server 2022 and VyOS routers by yourself. 

Each VyOS router needs at least two interfaces, one for the internal network (with a host-only VMnet) and another for WAN purposes (Using a VMnet with Bridge mode). In VMware go to `Edit` then click on `Vitual Network Editor`. You need to configure two VMnets (in this case, VMnet0 for Bridge adapter and VMnet1 for host-only). Then you have to go to your router's settings and add another Network adapter. Set each adapter to the correct VMnet. Do this for all routers. (the images below show VyOS-KBL's VMnet setup)
<p align="center">
  <img src="images/004a.vmnets.png" width="350">
  <img src="images/004b.kblvmnet.png" width="350">
</p>

Now you can configure your router's interfaces. In order to apply changes to VyOS interfaces you need to go to 'configure' mode by using `configure` command. Then use this command `set interfaces ethernet eth0 address x.x.x.x/x` to manually set IP Address of a router's interface. Use `set interfaces ethernet eth1 address dhcp` for the bridged interfaces of the router to get an IP from the Bridge Network of your computer. You can optionally use `set interfaces ethernet eth0 description To-KBL` to set a description for your interface. Don't forget to commit and save your changes by `commit` and `save` commands. You can run `show interfaces` command to list the interfaces. Image below shows the interface config for the VyOS-KBL:

![kbl-interfaces](images/004-kblvyosinterfaces.png)

Configure network of the servers existing in each site.
<p align="center">
  <img src="images/001%20-%20ip%20set%20site%20kabul.png" width="250">
  <img src="images/002%20-%20ip%20set%20site%20herat.png" width="250">
  <img src="images/003%20-%20ip%20set%20site%20mazar.png" width="250">
</p>

In order to have reachability to each site, we need to configure a routing protocol. In this case we are using OSPF. Configure OSPF on each site's VyOS router following the commands in the image below. 

![ospf-config](images/005-ospf-config.png)

`set protocols ospf redistribute connected` tells VyOS to advertise all directly connected networks (that are not already included via network statements) into OSPF. You can check OSPF neighborship with this command `show ip ospf neighbors`, use `run` with to excute non-configuration commands in configuration mode. Image below shows neighbors associated with KBL router (you can see that it has only one neighbor which is VyOS-MZR. HRT's router hasn't been configured with OSPF yet)

![ospf-show](images/005-ospf-nei.png)

You can set the DNS server of the router using `set system nameserver x.x.x.x` command. You can optionally set a defualt route using your Bridge Network's gateway as the next-hop for the default route.

![def-route](images/006b-default0-route.png)

Another important configuration is Network Address Translation (NAT) for the end-hosts in each site to have internet access. Follow below commands to configure NAT in each router:

![nat](images/006a-nat.png)

We basically used PAT in this case which is achieved using `set nat source rule 100 translation address masquerade`.
You can verify NAT translations with `show nat source translations` command. After NAT is configured, make sure end hosts have internet access by pinging google.com.

<p align="center">
  <img src="images/nattranslations.png" width="400">
  <img src="images/internetaccess.png" width="400">
</p>

---

# Site-to-Site IPsec VPN Configuration

In this section, we are going to configure site-to-site IPsec VPN between all three sites. We are specifically Route-Based VPN in this lab. For more information you can refer to VyOS documentation of Site-to-Site IPsec [here](https://docs.vyos.io/en/latest/configuration/vpn/ipsec/site2site_ipsec.html)

### IKE Phase 1 (ISAKMP Proposal):
Configure IKE phase 1 on all routers. Remember that the negotiated parameters in this phase must match across all the routers in all sites except for the `lifetime` parameter but it is recommended to use the same value for all peers. Do the same for all the other routers participating in the VPN configuration. Image below shows IKE phase 1 config for KBL router:

![IKE-Phase1](images/vpn/vpnIKEphase1.png)

### ESP Phase 2 (IPsec Proposal):
Configure ESP phase 2 on all routers. The parameters must match on all routers. Do the same for all peers:

![ESP-Phase2](images/vpn/vpnIKEphase2.png)

Specify interface facing to the protected destination. In this case eth3 on VyOS-KBL:

![VPN-Interface](images/vpn/vpnInterface.png)

### PSK Configuration:
Configure PSK keys and authentication ids for this key if authentication type is PSK. In our case, there are three peers participating in VPN configuration, so three IDs should be created. Then you need to specify the PSK secret key (same on all peers):

![PSJ](images/vpn/vpnPSKconfig.png)

### Configure Peers:
Configure peer and apply IKE-group and esp-group to each peer. Remember `connection type` values are either `initiate` or `respond` in this case. For the VyOS-KBL router, both connections to each peer is set to `initiate`. The `initiate` connection-type indicates that this peer starts the IPsec negotiation (sends the first IKE packet) and `respond` connection-type indicates that the peer waits for incoming IKE negotiations (this connection-type is configured in MZR and HRT sites for KBL peer). The `local-address` and `remote-address` define WAN IP address of the initiator and responder.

HRT Peer configuration:
![peer-hrt](images/vpn/vpnPeerHRT.png)
MZR Peer configuration:
![peer-mzr](images/vpn/vpnPeerMZR.png)

### VTI Tunnels configuration:
For Route-based VPN create VTI interfaces, set IP addresses to these interfaces and bind these interfaces to the VPN peers.

![VTI](images/vpn/vpnVti.png)

After this, you will need to route the overlay network. Delete the OSPF internal routes (192.168.1.0/24 for VyOS-KBL) and instead route this network and the tunnel network (10.0.0.0/24) using static routes. Remember to use different routing protocols in overlay and underlay to avoid recursive routing. Do this on all routers.

![overlay](images/vpn/overlay.png)

## Checking VPN Configuration:
confirm your VPN connection throughout all sites using `show vpn ike sa` and `show vpn ipsec sa`. Images below show IKE and IPsec establishments for VyOS-HRT:

![ike-sa](images/vpn/vpnIkeSa.png)
And
![ipsec-sa](images/vpn/vpnIpsecSa.png)

---

# AD DS Installation and Configuration
As part of this lab, We are going to simulate a real-world enterprise network with multiple physical locations and domain controllers.

### 🏷️ Domain Information

- **Domain Name**: `project.lab`
- **Sites**:
  - KBL (main site) --> KBL-DC is the PDC
  - HRT (branch site) --> HRT-DC is the ADC
  - MZR (branch site) --> MZR-DC is the RODC

### AD DS installation:
Inside the Server Manager's Dashboard, go to `Add roles and features`. You'll be prompted with a wizard to install roles and features for the server. Follow the basic steps, in `Server Roles` section select two roles "Active Directory Domain Services" and "DNS Server". Select Next for the other steps and click install at the end:

![ADDS-Install](images/AD/addsInstall.png)

After installation of AD DS, you need to promote the server to a domain controller. Click on `promote this server to a domain controller` on the AD DS installation wizard after installation is done. You'll be prompted with AD DS Configuration Wizard. In the `Deployment Configuration` select "Add a new forest" and then name your root domain (in this case = project.lab). This will indicate that this server will be configured as Primary Domain Controller (PDC). Select Next for other steps and you'll be done with DC promotion.

![KBL-Promo](images/AD/addsPromo.png)

For the HRT-DC, install the AD DS and DNS roles, then promote it. In the `Deployment Configuration` of HRT-DC's configuration wizard, select `Add a domain controller to an existing domain` and provide the name of your previously created domain (project.lab). Click Next for the other steps.

![HRT-Promo](images/AD/addsHRTpromo.png)

Don't forget to set the Primary DNS of HRT-DC to itself (192.168.2.20 in this case). You can choose to set the alternate DNS server to KBL-DC's IP address. Do this on MZR-DC as well.

![set-IP-After-Promo](images/AD/setIpHRTpromo.png)

Follow the same step for MZR site. Install the AD DS role and then add it to your existing domain. Remember that in our case, MZR-DC is configured to be a Read-Only Domain Controller (RODC). You can find out more about what a RODC is in [here](https://learn.microsoft.com/en-us/windows/win32/ad/rodc-and-active-directory-schema):

<p align="center">
  <img src="images/AD/addsMZRpromo.png" width="400">
  <img src="images/AD/addsMZRrodc.png" width="400">
</p>

---

### DNS Configuration:
Configure `Forward Lookup Zone` and `Reverse Lookup Zone` of each server.

![DNS](images/AD/dns.png)

---

### Configuring Sites:
We need to determine our sites by creating sites in the Active Directory Sites and Services.

![Sites-Services](images/AD/adss.png)

Click on `Sites`, you will see a site called "Default Site", right click on it and rename to the main site (KBL-Site). Right click on `Sites` and choose "New Site" to create a new site, do this for HRT and MZR sites.

![create-Sites](images/AD/createSites.png)

Then we need to create subnet for each site. Subnets determine the range of IP addresses used for each site. Go to `Sites` and then `Subnets`, right click on Subnets and choose `new subnet`. Create subnet for each site and configure the IP addresses assessed for each site:

![Subnets](images/AD/sitesSubnet.png)

Put each site's DC to its coresponding site. You can drag and drop the servers to each specified site's servers folder:

![Set-Servers](images/AD/serverSet.png)

Next, create site links between the sites. An inter-site link is a logical connection in Active Directory that represents the network path between two or more AD sites. It is used by Active Directory replication to transfer changes between domain controllers (DCs) that are in different physical or logical sites. By default, Active Directory inter-site replication happens every 180 minutes. You can change this by going to a site link's properties.

<p align="center">
  <img src="images/AD/siteLink.png" width="400">
  <img src="images/AD/sitesInterSite.png" width="400">
</p>

### Replication Test
We can check if replication works through sites. For this we first create an Organizational Unit (OU) inside Active Directory Users and Computers and name it IT. Then we create a user "IT01" inside the OU and see if this is also replicated on HRT-DC and MZR-DC servers:

![OU](images/AD/createOU.png)

Then go to AD Sites and Services and follow below steps, image below shows replication test for HRT-DC, do the same for MZR-DC:

![Replica](images/AD/replicaTest.png)

Finally check the HRT-DC and MZR-DC servers if they have recieved the mentioned OU.

<p align="center">
  <img src="images/AD/replicaHRT.jpg" width="400">
  <img src="images/AD/replicaMZR.png" width="400">
</p>

---
## 🔧 Step-by-Step Setup
2. Splunk Enterprise Setup:
   I started by downloading both Splunk Enterprise and Ubuntu Server. Splunk will be hosted on the Ubuntu Server virtual machine.
 📌 Introduction to Splunk Enterprise:
 Splunk Enterprise is a widely adopted platform designed for collecting, indexing, and analyzing machine data in real time. It supports log ingestion from a wide range of sources and is commonly used in Security Operations Centers (SOCs) as a Security Information and Event Management (SIEM) solution. With its powerful search capabilities and rich visualizations, Splunk helps security teams detect threats, investigate incidents, and monitor system activity effectively.
2)my benefits from working on splunk on this lab.
Integrating Splunk Enterprise into my home lab environment has been a valuable step toward building practical cybersecurity skills. With this setup, I can continuously monitor system activity, detect suspicious behavior, and simulate real-world SOC operations. This hands-on approach not only strengthens my understanding of how modern security tools work but also improves my ability to respond to potential threats—an essential skillset for anyone pursuing a career in cybersecurity.
⬇️ Downloading Splunk Enterprise (Free Trial)
1.Visit the official [Splunk website](https://www.splunk.com)
2.Log in or create a free account.
3.Navigate to Products > Free Trials and Downloads.
4.Select Splunk Enterprise, then click Get My Free Trial.
5.Choose your host operating system. Since I’m using Ubuntu Server, I selected the Linux .deb package for Debian-based distributions.

2.Downloading Ubuntu server:
 I downloaded the latest LTS version of Ubuntu from the [Ubuntu Downloads page](https://ubuntu.com/download/server). At the time of writing, the newest version available was Ubuntu 24.04.2 LTS, and the ISO file was approximately 3GB

   3.Creating the Ubuntu VM on VMware:
With the ISO downloaded, I proceeded to create the Ubuntu VM on vmware workstation:
-Allocated 2 CPUs, 4 GB RAM, and 30 GB disk space.
-Used VMnet2(hostonly) to ensure connectivity with my vyos router.
![ubentuServerVMOverview.png](images/sec/ubentuServerVMOverview.png)
  Network Configuration:
![Image of VMnet2 config](images/sec/VMnet2.png)

4. Installing Uebntu Server:
I attached the ISO to the new VM and started the system. I used the default installation options, pressing Enter until installation completed and the system rebooted.
5. Enabling Remote Access with PuTTY
After installation, I used PuTTY on my Windows host to connect to the Ubuntu Server VM.
🔌 Prerequisites
The VM must be powered on and connected to the same network as the host.
Ensure OpenSSH Server is installed (enabled by default). If not, install it:
<pre lang="markdown">bash
  sudo apt install openssh-server</pre>
🔍 Finding the IP Address
Run this command on the Ubuntu VM:
<pre lang="markdown">bash
  ip a</pre>
📸 Example:
![ipaddress config of ubentu server](images/sec/ipA.png)
💻 Access via PuTTY
Open PuTTY on Windows.
Enter the IP address in the Host Name field.
Set Port to 22 and Connection Type to SSH.
Click Open.
📸 PuTTY Configuration:
![putty](images/sec/putty.png)
Log In
A terminal window will open
Enter your Ubuntu username and password
![Login Putty](images/sec/loginPutty.png)
![Putty Overview](images/sec/puttyOverview.png)
   

I then took a snapshot of the VM at this clean, fully updated state to serve as a baseline for future rollback if needed.

7) uentu server network config:
   Since I'm working in the HRT-Site environment, I assigned the following static IP settings:
-IP Address: 192.168.2.10
-Gateway: My VyOS router
-DNS Server: My home router's IP (instead of 8.8.8.8, since external DNS is filtered in my country)
⚠️ Note: Setting DNS to 8.8.8.8 caused internet loss due to regional filtering. Using my home router as DNS resolved the issue.
🛠️ To Set Static IP
Edit the Netplan configuration file:
<pre lang="Markdown">bash
  sudo nano /etc/netplan/00-installer-config.yaml</pre>
📸 Netplan Configuration:
   ![network config ubentu server](images/sec/netplan.png)
   Apply changes and verify internet connectivity:
   <pre lang="markdown"> bash 
   sudo netplan apply
   ping google.com </pre>
   📸 Ping Test:
 ![ping](images/sec/ping.png)
7. System Update
Finally, I updated the Ubuntu system to ensure all packages were current:
<pre lang="markdown">bash
sudo apt update
sudo apt upgrade</pre>


8.⚙️ Installing VMware Tools on Ubuntu Server
Installing VMware Tools improves virtual machine performance and enables features such as better network drivers, time synchronization, and smoother integration with the host system.
and aslo we need to share splunk setup from host machine wit the vm(ubentu server).
For Ubuntu Server, the recommended method is to install the open-vm-tools package provided through the official Ubuntu repositories.
📦 Step-by-Step Guide
Update Package List
<pre lang="Markdown">
bash
sudo apt update
</pre>
Install Open VM Tools 
<pre lang="Markdown">
bash
sudo apt install open-vm-tools -y
</pre>
Verify the Installation
Check the version to confirm installation:
<pre lang="Markdown">
bash
vmware-toolbox-cmd -v
</pre>
[vmTool](images/sec/vmTool)
📌 Notes
There's no need to mount the ISO or install VMware Tools manually—the open-vm-tools package is the official and preferred method for Ubuntu.
Now that VMware Tools is installed, your VM will perform better, and features like time sync and graceful shutdown will work reliably.

🔄 Sharing Splunk Installer from Host to Ubuntu Server VM
To transfer the Splunk Enterprise installer from your Windows host machine to your Ubuntu Server VM, you can use a VMware feature called Shared Folders. This allows your virtual machine to access a directory from your host system as if it were part of the VM's file system.

🧰 What is Shared Folders?
Shared Folders is a VMware Workstation/Player feature that enables file sharing between the host and guest systems without using USB drives, SCP, or network file transfers. It's ideal for quickly moving files like installers, scripts, or configuration files.

📦 Step-by-Step Guide to Enable Shared Folders
🔒 Ensure VMware Tools (open-vm-tools) is installed and running on your Ubuntu Server before proceeding.
1. Configure Shared Folder in VMware
-Power off your Ubuntu VM (if it's running).
-Open VMware Workstation or Player.
-Select your VM > Click Edit virtual machine settings.
-Go to the Options tab.
-Select Shared Folders.

Example:

![shareFolderConfig](images/sec/shareFolderConfig.png)
-Choose Always enabled or Enabled until next power off or suspend.
-Click Add… and:
-Browse to the folder on your host where the Splunk installer (.deb file) is located.
-Give it a name like splunk-share.
-Click Finish, then OK to apply the changes.
![shreFolderBrowse](images/sec/shareFolderBrowse.png)
-Start your Ubuntu VM.

2. Access the Shared Folder in Ubuntu
   
Once the VM is running, VMware automatically mounts the shared folder to the path:

<pre lang="Markdown">bash
cd /mnt/hgfs
</pre>

To view the shared folder:

<pre lang="Markdown">bash
  ls /mnt/hgfs
</pre>

and if you couldn't see your shared folder after ruuning the above command, try this command first

<pre lang="Markdown">bash
  sudo vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other
</pre>

  This command mounts the shared folder from the VMware host system into your Ubuntu Server VM, using VMware’s FUSE-based file system.

You should see your shared folder name(e.g.,project.lab):

![sharedFolder](images/sec/sharedFolder.png)

access the shared folder

<pre lang="Markdown">bash
  cd /mnt/hgfs/splunk-share
</pre>

Now you can access the Splunk installer from there and install it:

<pre lang="Markdown">bash
sudo dpkg -i splunk-*.deb
</pre>

![splunkInstallation](images/sec/splunkInstallation.png)

🛠️ You may need to run sudo apt --fix-broken install if dependencies are missing.

now change to the direcorty where splunk is:

<pre lang="Markdown">bash
  cd /opt/splunk
</pre>

and list it's contains:

<pre lang="Markdown">bash
  ls -la
</pre>

![splunkConfig1](images/sec/splunkConfig1.png)

this shows that splunk is installed successfully

now hit to the follwing directory and run the follwing commands which will run splunk:

<pre lang="Markdown">bash
cd /bin/splunk
  sudo ./splunk start
</pre>

go through the liesanse aggrement and you will be asked to a username and password. 

now we want to enable splunk on boot so every time the vm boots up splunk will be running:

<pre lang="Markdonw">bash
  cd bin
  sudo ./splunk enable boot-start
</pre>

![splunkEnabledonBootStart](images/sec/splunkEnabledonBootStart.png)

the next step is to install splunk forwarder on our windows machines. I am going to demostrate how I did it on windows 10. It should be the same accross other vms too. 
First we need to join my windows mashine to the domain. To do so will asign a static ip address to it and check it's reachblitiy with the DC.

My windows 10's network adapter is in vmnet2(Hostonly) which is the same vmnet in which my ubentu server, vyos router and kali lunix are.

![ipconfigWindows10](images/sec/ipconfigWindows10.png)

I joined my windows 10 vm to project.lab domain and also renamed it to HRT-PC01

![ipconfigHRT-PC01](images/sec/ipconfigHRT-PC01.png)

![welcomeToProject.lab](images/sec/welcomeToProject.lab.png)

afterwards you pc will require a reboot and then log in as a domain user:

![loginAsAdmin](images/sec/loginAsAdmin.png)


now my vm is a part of project.lab domain

![aPartofDomain](images/sec/aPartofDomain.png)

📦 What is Splunk Universal Forwarder?
The Splunk Universal Forwarder (UF) is a lightweight version of Splunk designed specifically for collecting and forwarding logs and machine data to a central Splunk instance (typically a Splunk Enterprise or Splunk Cloud server). It runs as a background service and is optimized for minimal resource usage while maintaining high-speed data transfer.

Unlike the full Splunk Enterprise instance, the Universal Forwarder has no web interface or data indexing capabilities. Its primary role is to gather data (e.g., Windows Event Logs, files, registry, or performance metrics) from the host and send it to a central Splunk indexer or heavy forwarder.

🧠 Why Use Splunk Forwarder?
Using a Universal Forwarder in a distributed Splunk architecture offers several key benefits:

Centralized Logging: Collect logs from multiple endpoints and forward them to a centralized Splunk server for analysis and visualization.

Lightweight & Efficient: Designed to run with minimal system impact, making it ideal for desktops, servers, or embedded systems.

Secure Data Transmission: Supports encrypted and authenticated communication with your Splunk indexers.

Real-time Data Delivery: Enables near real-time forwarding of logs, which is crucial for security monitoring, incident response, and compliance.

In this documentation, we will walk through the steps to install and configure the Splunk Universal Forwarder on a Windows 10 machine and ensure it forwards logs to a Splunk Enterprise server running on a different system (in our case, an Ubuntu server).

📥 Step 1: Download and Install the Forwarder
Download the 64-bit MSI installer for Windows.
head to the [splunk website](www.splunk.com)
 




   
