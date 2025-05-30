# **Cybersecurity Lab Setup Guide: Building Your Virtual Pentesting & Red Teaming Environment**

This guide provides a highly detailed, step-by-step walkthrough for establishing a robust and segmented virtual cybersecurity lab using KVM/Qemu on your Arch Linux host. It's designed to be a comprehensive and actionable resource, ensuring you can perfectly set up environments for Web Application Security, Network Security, and Active Directory Security, complete with a pfSense firewall and dedicated analysis zones.  
**Lab Goals:**

* **Web Application Security:** Hands-on practice with the OWASP Top 10 vulnerabilities using intentionally insecure web applications.  
* **Network Security:** Develop skills in reconnaissance, enumeration, exploitation, and post-exploitation on diverse vulnerable Linux and Windows targets.  
* **Active Directory Security:** Simulate enterprise environments to master privilege escalation, lateral movement, and persistence techniques within a Windows domain.  
* **MAL network:** Provide a secure and isolated environment for malware analysis, reverse engineering, and exploit development.

**Your Host System Resources:**

* **Operating System:** Arch Linux  
* **Virtualization:** KVM/Qemu  
* **Memory:** 32GB RAM \+ 32GB Swap  
* **Storage:** 1TB dedicated for the lab

## **1\. Prerequisites: Prepare Your Arch Linux Host**

Before you begin creating any virtual machines, it is crucial to ensure your Arch Linux host is correctly configured and optimized for KVM/Qemu virtualization.

### **1.1. Verify Hardware Virtualization Support**

Hardware virtualization extensions (Intel VT-x or AMD-V) are essential for KVM to function efficiently, allowing virtual machines to execute instructions directly on the CPU with near-native performance. These extensions must be supported by your CPU and enabled in your system's BIOS/UEFI settings.

* **Check CPU Support:** Open a terminal and run:  
  egrep \-c '(vmx|svm)' /proc/cpuinfo

  * **Expected Output:** A number greater than 0 (e.g., 4, 8, 16). This indicates that your CPU supports virtualization.  
  * **Troubleshooting:** If the output is 0 or blank, hardware virtualization is either not supported by your CPU or, more commonly, not enabled in your system's BIOS/UEFI settings. You **must** reboot your system, enter your BIOS/UEFI settings (often by pressing Del, F2, F10, or F12 during boot), and enable options such as "Intel Virtualization Technology," "Intel VT-x," "AMD-V," or "SVM Mode." Save changes and reboot. Without this, KVM will not function correctly.

### **1.2. Install KVM/Qemu and Management Tools**

Install the core virtualization components and virt-manager, a graphical user interface that simplifies VM creation and management, which is highly recommended for ease of use.  
sudo pacman \-S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat

* **Explanation of Packages:**  
  * qemu: The core emulator and virtualizer.  
  * virt-manager: A powerful graphical tool for managing KVM/Qemu VMs.  
  * virt-viewer: A simple client for viewing VM consoles.  
  * dnsmasq: Often used by libvirt for DHCP and DNS services on default virtual networks (though pfSense will handle DHCP for your custom networks).  
  * vde2, bridge-utils, openbsd-netcat: Utilities for network bridging and other networking functionalities.

### **1.3. Configure User Permissions**

To manage virtual machines and interact with the libvirt daemon without needing sudo for every command, add your user account to the libvirt and kvm groups. This adheres to the principle of least privilege, enhancing security.  
sudo usermod \-aG libvirt,kvm $(whoami)

* **Crucial Step:** For these group changes to take effect, you **must log out of your Arch Linux session and log back in (or reboot)**. If you skip this, virt-manager might not connect to libvirtd or you might encounter permission errors.

### **1.4. Start and Enable Libvirt Service**

The libvirtd service is the daemon that manages KVM/Qemu virtual machines. It must be running and enabled to start automatically on boot for virt-manager and virsh commands to function correctly.  
sudo systemctl enable \--now libvirtd

* **Verify Service Status:** Confirm that libvirtd is active and running:  
  sudo systemctl status libvirtd

  * **Expected Output:** Look for Active: active (running). If it's not running, check journalctl \-xe for error messages.  
* **Verify KVM Modules:** Ensure the KVM kernel modules are loaded:  
  lsmod | grep kvm

  * **Expected Output:** You should see kvm\_intel (for Intel CPUs) or kvm\_amd (for AMD CPUs) listed. If not, ensure hardware virtualization is enabled in BIOS/UEFI and try sudo modprobe kvm\_intel (or kvm\_amd).

### **1.5. Disk Image Format (Qcow2)**

When creating virtual machine disk images, qcow2 is the native and highly recommended format for KVM. virt-manager typically uses this by default, but it's good to be aware of its benefits.

* **Key Benefits of qcow2:**  
  * **Thin Provisioning:** Disk space is allocated on demand. The virtual disk file starts small and grows only as data is written by the VM, saving physical storage.  
  * **Snapshots:** Essential for a pentesting lab\! qcow2 allows you to easily create and revert to previous states of your VMs, enabling quick resets of vulnerable targets.  
  * **Cloning:** Efficiently create copies of existing VMs without duplicating the entire base image.

### **1.6. Enable Nested Virtualization (Recommended)**

Nested virtualization allows you to run a hypervisor *inside* one of your virtual machines (e.g., running Hyper-V within a Windows VM, or a nested KVM instance). While not strictly necessary for every VM in your lab, it provides maximum flexibility for advanced scenarios (e.g., running WSL2 or Docker Desktop on Windows VMs, or experimenting with nested hypervisors).

* **For Intel CPUs:**  
  echo "options kvm-intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf

* **For AMD CPUs:**  
  echo "options kvm-amd nested=1" | sudo tee /etc/modprobe.d/kvm-amd.conf

* **Apply Changes:** For the changes to take effect, you need to unload and reload the KVM module. If you have any KVM VMs running, you **must shut them down first**.  
  * **Option 1 (Recommended):** Reboot your Arch Linux host. This is the simplest and most reliable way to apply kernel module options.  
  * **Option 2 (Without reboot, requires no KVM VMs running):**  
    sudo modprobe \-r kvm\_intel \# (or kvm\_amd for AMD)  
    sudo modprobe kvm\_intel nested=1 \# (or kvm\_amd nested=1 for AMD)

## **2\. Network Setup: Building Your Lab Segments with pfSense**

The cornerstone of your secure and effective lab is meticulous network segmentation, with pfSense acting as your central router and firewall. You will create distinct virtual network bridges on your Arch Linux host, and then connect pfSense's virtual network interfaces to these bridges.  
**Network Diagram Reference:**  

```plaintext
                                     +---------------------+  
                                     |     Internet /      |  
                                     |   Home Router (DHCP)|  
                                     +----------+----------+  
                                                |  
                                                | (Physical NIC / Bridge: br-wan)  
                                                |  
                                     +----------+----------+  
                                     |   Arch Linux Host   |  
                                     |     (KVM/Qemu)      |  
                                     +----------+----------+  
                                                |  
                                                | (Virtual Bridge: br-wan)  
                                                |  
                                     +----------+----------+  
                                     |   pfSense Firewall  |  
                                     | (Virtual Machine)   |  
                                     |---------------------|  
                                     | WAN Interface       |  
                                     | (DHCP from Home Rtr)|  
                                     | LAN Interface       |  
                                     | DMZ Interface       |  
                                     | MAL Interface       |  
                                     +----------+----------+  
                                                |  
          +-------------------------------------+-------------------------------------+  
          |                                     |                                     |  
          | (Virtual Bridge: br-lan)            | (Virtual Bridge: br-dmz)            |  
          |                                     |                                     |  
+---------+-------------------------+ +-----------+-----------------------+ +-----------+-----------------------+  
|           LAN Network             | |           DMZ Network             | |     MAL Network                   |  
|         (10.0.0.0/24)             | |        (172.16.0.0/24)            | |       (192.168.100.0/24)          |  
+---------+-------------------------+ +-----------+-----------------------+ +-----------+-----------------------+  
          |                                     |                                     |  
          |                                     |                                     |  
+---------+----------+              +-----------+-----------+             +-----------------------------------+  
| Kali Linux         |              | Docker Host VM        |             | +-----------+-----------+         |  
| (Attacker Machine) |              | (e.g., Ubuntu Server) |             | | REMnux VM (Linux)     |         |  
+--------------------+              |                       |             | | (Docker Container Host)|         |  
          |                         |  +-----------------+  |             | +-----------------------+         |  
          |                         |  | OWASP Juice Shop|  |             | |  +-----------------+  |         |  
          |                         |  | DVWA            |  |             | |  | Malware Samples |  |         |  
+--------------------+              |  | WebGoat         |  |             | |  | Exploit Code    |  |         |  
| Windows Server 2019|              |  +-----------------+  |             | |  +-----------------+  |         |  
| (Domain Controller)|              +-----------------------+             | +-----------------------+         |  
| (10.0.0.10 - Static)|                                                    |                                   |  
+--------------------+                                                    | +-----------------------+         |  
          | (Active Directory Domain: pentestlab.local)                   | | FlareVM (Windows)     |         |  
          |                                                               | | (Malware Analysis)    |         |  
+--------------------+                                                    | | (DHCP from pfSense)   |         |  
| Windows 10 Clients |                                                    | +-----------------------+         |  
| (Domain-joined)    |                                                    +-----------------------------------+  
| (DHCP from pfSense)|  
+--------------------+  
          |  
+--------------------+  
| Metasploitable 3   |  
| (Windows Build)    |  
| (DHCP from pfSense)|  
+--------------------+  
          |  
+--------------------+  
| Metasploitable 3   |  
| (Ubuntu Build)     |  
| (DHCP from pfSense)|  
+--------------------+  
```

### **2.1. Create Virtual Bridges on Arch Linux Host**

You will create four distinct virtual bridges. The br-wan bridge will be a **physical bridge**, directly connecting to your host's physical network adapter. This allows pfSense to obtain an IP address from your home router, simulating internet access. The other bridges (br-lan, br-dmz, br-isolated) will be **isolated virtual networks**. VMs connected to these isolated bridges can only communicate with other VMs on the *same* bridge, or with pfSense, providing crucial security and containment for your lab environment.  
We'll use systemd-networkd for configuring the br-wan bridge, as it's robust for physical network configurations. For the isolated virtual networks, libvirt's built-in network definitions are generally simpler to manage via virt-manager or virsh.  
First, identify your physical network interface:  
Open a terminal and run ip a. Look for your active Ethernet interface (e.g., enp0s31f6, eth0, eno1). Replace YOUR\_PHYSICAL\_NIC with its actual name in the steps below.

#### **a. br-wan (WAN Interface \- Bridged to Physical NIC)**

This bridge will connect pfSense's WAN interface to your physical home network, allowing it to get an IP from your home router (simulating internet access).

1. **Create the bridge definition file (.netdev):**  
   sudo tee /etc/systemd/network/10-br-wan.netdev \<\<EOF  
   \[NetDev\]  
   Name=br-wan  
   Kind=bridge  
   EOF

2. **Configure your physical NIC to be part of the bridge (.network):**  
   sudo tee /etc/systemd/network/20-br-wan-if.network \<\<EOF  
   \[Match\]  
   Name=YOUR\_PHYSICAL\_NIC \# \<--- IMPORTANT: REPLACE THIS with your actual physical NIC name\!  
   \[Network\]  
   Bridge=br-wan  
   EOF

3. **Restart systemd-networkd to apply changes:**  
   sudo systemctl restart systemd-networkd

   * **Troubleshooting br-wan:** If br-wan doesn't come up or your host loses network connectivity, double-check the YOUR\_PHYSICAL\_NIC name. Ensure no typos. Also, review sudo journalctl \-xe | grep systemd-networkd for specific errors. Sometimes, a full host reboot is needed for systemd-networkd changes to apply cleanly.  
4. **Allow Qemu to use the bridge:** This step is crucial for KVM/Qemu to be able to attach VMs to this bridge.  
   echo 'allow br-wan' | sudo tee /etc/qemu/bridge.conf

   * **Troubleshooting qemu/bridge.conf:** If you get permission errors when trying to connect a VM to br-wan in virt-manager, ensure this file exists and has the correct content.

#### **b. br-lan (LAN Network \- Isolated Virtual Network)**

This bridge will host your internal lab network (10.0.0.0/24). VMs on this network will receive IPs from **pfSense's DHCP server**.

1. **Create the libvirt network XML definition:**  
   tee /tmp/br-lan.xml \<\<EOF  
   \<network\>  
     \<name\>br-lan\</name\>  
     \<bridge name="br-lan"/\>  
     \<forward mode='isolated'/\>  
     \<ip address='10.0.0.1' prefix='24'\>  
     \</ip\>  
   \</network\>  
   EOF

2. **Define, start, and autostart the libvirt network:**  
   virsh net-define /tmp/br-lan.xml  
   virsh net-start br-lan  
   virsh net-autostart br-lan

   * **Troubleshooting libvirt networks:** If virsh net-define fails, check for XML syntax errors or if a network with the same name already exists. If virsh net-start fails, check sudo journalctl \-xe | grep libvirtd for errors.

#### **c. br-dmz (DMZ Network \- Isolated Virtual Network)**

This bridge will host your Demilitarized Zone (DMZ) network (172.16.0.0/24). VMs on this network will receive IPs from **pfSense's DHCP server**.

1. **Create the libvirt network XML definition:**  
   tee /tmp/br-dmz.xml \<\<EOF  
   \<network\>  
     \<name\>br-dmz\</name\>  
     \<bridge name="br-dmz"/\>  
     \<forward mode='isolated'/\>  
     \<ip address='172.16.0.1' prefix='24'\>  
     \</ip\>  
   \</network\>  
   EOF

2. **Define, start, and autostart the libvirt network:**  
   virsh net-define /tmp/br-dmz.xml  
   virsh net-start br-dmz  
   virsh net-autostart br-dmz

#### **d. br-isolated (MAL Network \- Strictly Isolated Virtual Network)**

This bridge will host your dedicated malware analysis/exploit development VMs (192.168.100.0/24). VMs on this network will receive IPs from **pfSense's DHCP server**.

1. **Create the libvirt network XML definition:**  
   tee /tmp/br-isolated.xml \<\<EOF  
   \<network\>  
     \<name\>br-isolated\</name\>  
     \<bridge name="br-isolated"/\>  
     \<forward mode='isolated'/\>  
     \<ip address='192.168.100.1' prefix='24'\>  
     \</ip\>  
   \</network\>  
   EOF

2. **Define, start, and autostart the libvirt network:**  
   virsh net-define /tmp/br-isolated.xml  
   virsh net-start br-isolated  
   virsh net-autostart br-isolated

#### **e. Verify Bridge Creation**

After creating all bridges, confirm that they are active and correctly configured on your Arch Linux host.  
ip a | grep br-  
brctl show  
virsh net-list \--all

* **Expected Output:** You should see br-wan, br-lan, br-dmz, and br-isolated listed. For br-wan, brctl show should also indicate that your physical NIC is an interface of br-wan. If any are missing or not active, re-check the previous steps and journalctl \-xe.

### **2.2. Install and Configure pfSense VM**

pfSense will serve as the central router and firewall for your entire lab, controlling all traffic flow between your segmented networks and to/from the internet.

* **Download pfSense ISO:** Obtain the latest stable CE (Community Edition) ISO image from the official pfSense website ([https://www.pfsense.org/download/](https://www.pfsense.org/download/)).  
1. **Create pfSense VM in virt-manager:**  
   * Open virt-manager (type virt-manager in your terminal).  
   * Click "File" \-\> "New Virtual Machine."  
   * **Step 1 (Choose how you would like to install the operating system):** Select "Local install media (ISO image or CDROM)." Click "Forward."  
   * **Step 2 (Choose ISO or CDROM):** Click "Browse..." \-\> "Browse Local" and navigate to your downloaded pfSense ISO.  
   * **Step 3 (Choose the operating system you are installing):** Type FreeBSD and select "FreeBSD 13.x (or latest available)."  
   * **Step 4 (Memory and CPU):**  
     * **Memory (RAM):** 4096 MB (4 GiB) \- This provides a good balance for pfSense's features.  
     * **CPUs:** 2 \- Sufficient for routing and firewalling in a lab environment.  
   * **Step 5 (Enable storage for this virtual machine):**  
     * **Disk Size:** 50 GiB (Ensure "Select or create custom storage" is chosen, and specify 50 GiB for the qcow2 image).  
   * **Step 6 (Name the virtual machine and configure the installation):**  
     * **Name:** pfSense  
     * **Crucial \- Network Selection (Initial):** By default, it might select Default (NAT). **Do NOT click "Finish" yet.**  
     * Check the box "Customize configuration before install." Click "Finish."  
2. Configure pfSense Network Interfaces in virt-manager:  
   After clicking "Finish," the VM details window will open. This is where you connect pfSense's virtual NICs to your newly created host bridges.  
   * Locate the existing "NIC: Virtual network interface" (it will likely be connected to Default). Click "Remove Hardware" to remove this default NIC.  
   * Click "Add Hardware" \-\> "Network."  
     * **Network source:** Select "Bridge device" from the dropdown.  
     * **Device name:** Choose br-wan.  
     * **Device Model:** Set to virtio (for best performance).  
     * Click "Finish."  
   * Repeat "Add Hardware" \-\> "Network" for the remaining interfaces, ensuring each is connected to the correct bridge:  
     * **NIC 2 (LAN):** Network source: "Bridge device", Device name: br-lan, Device Model: virtio.  
     * **NIC 3 (DMZ):** Network source: "Bridge device", Device name: br-dmz, Device Model: virtio.  
     * **NIC 4 (MAL):** Network source: "Bridge device", Device name: br-isolated, Device Model: virtio.  
   * **Important:** Ensure all NICs are set to virtio for optimal performance and compatibility with pfSense.  
   * **Crucial: Qemu Machine Type for pfSense:** Due to known compatibility issues between newer Qemu machine types and FreeBSD (which pfSense is based on), it's highly recommended to set an older, compatible machine type. Go to the "Overview" section of the VM details, click "Hypervisor Details," and change the "Chipset" or "Machine type" to an older, stable version, such as pc-q35-2.6 or similar. Failure to do so might lead to installation failures or boot issues.  
3. **Install pfSense:**  
   * Click "Begin Installation" in virt-manager.  
   * Follow the on-screen prompts for pfSense installation.  
   * **Interface Assignment (Crucial during pfSense setup):** During installation, pfSense will detect the virtual NICs (e.g., vtnet0, vtnet1, vtnet2, vtnet3 corresponding to your virtio NICs). You will be prompted to assign them to their respective roles:  
     * **WAN:** Assign the interface connected to br-wan (e.g., vtnet0).  
     * **LAN:** Assign the interface connected to br-lan (e.g., vtnet1).  
     * **DMZ:** Assign the interface connected to br-dmz (e.g., vtnet2).  
     * **MAL:** Assign the interface connected to br-isolated (e.g., vtnet3).  
     * Confirm the assignments.  
   * Complete the installation process and reboot the pfSense VM.  
4. Configure pfSense Post-Installation (via Console and Web UI):  
   After pfSense reboots, you will perform initial network configuration via its console (within virt-manager). All subsequent, more detailed configurations, including enabling DHCP servers and setting firewall rules, will be done via its web-based Graphical User Interface (GUI).  
   * **Initial Console Configuration (within virt-manager):**  
     * **WAN Interface:** This interface should be configured as a **DHCP client** by default, receiving an IP address from your home router. Verify it has successfully obtained an IP address.  
     * **LAN Interface:** Set a **static IP address:** 10.0.0.1 and Subnet mask: 255.255.255.0 (or /24). (You can also set static IPs for DMZ and MAL interfaces via the console at this stage, or later via the web GUI).  
   * **Web GUI Configuration (from Kali Linux VM \- No Internet Access Required for Kali for this step):**  
     * Open a web browser on your Kali Linux VM (or any VM connected to the br-lan network). **Note: Your Kali VM does NOT need internet access to perform this step, as it is communicating directly with pfSense on the internal br-lan network.**  
     * Navigate to the IP address of your pfSense LAN interface: https://10.0.0.1  
     * Proceed past any certificate warnings (as pfSense uses a self-signed certificate by default).  
     * Log in with the default credentials: admin / pfsense (it is highly recommended to change these immediately\!).  
     * **LAN Interface:**  
       * (Assuming you set its static IP via console)  
       * **Enable DHCP Server:** Navigate to **Services \-\> DHCP Server \-\> LAN** in the pfSense web GUI. Enable the DHCP server and configure an IP address range for client leases (e.g., 10.0.0.100 to 10.0.0.200). **pfSense will act as the DHCP server for all VMs on the LAN network.**  
     * **DMZ Interface:**  
       * Set a **static IP address:** 172.16.0.1  
       * Subnet mask: 255.255.255.0 (or /24)  
       * **Enable DHCP Server:** Go to **Services \-\> DHCP Server \-\> DMZ** in the pfSense web GUI. Enable it and configure a range (e.g., 172.16.0.100 to 172.16.0.200). **pfSense will act as the DHCP server for all VMs on the DMZ network.**  
     * **MAL Interface:**  
       * Set a **static IP address:** 192.168.100.1  
       * Subnet mask: 255.255.255.0 (or /24)  
       * **Enable DHCP Server:** Go to **Services \-\> DHCP Server \-\> MAL** in the pfSense web GUI. Enable it and configure a range (e.g., 192.168.100.100 to 192.168.100.200). **pfSense will act as the DHCP server for all VMs on the MAL network.**  
     * **Firewall Rules (Basic Setup):**  
       * **LAN to WAN:** By default, LAN should have outbound internet access.  
       * **DMZ to WAN:** Allow outbound internet access.  
       * **WAN to DMZ:** Allow inbound HTTP/HTTPS (ports 80, 443\) to your Docker Host VM's IP address (once known). This requires a NAT rule on pfSense.  
       * **DMZ to LAN:** **Block by default.** This is a critical security measure. Only allow specific, well-defined traffic if absolutely necessary for a lab exercise (e.g., if a DMZ service needs to query an internal database).  
       * **MAL Network:** By default, block all traffic to/from WAN, LAN, and DMZ. This network should be truly isolated for sensitive work, preventing any accidental egress of malware or exploit code.  
       * **Troubleshooting pfSense Connectivity:**  
         * **No Internet on VMs:** Check pfSense's WAN status (does it have an IP?). Check pfSense's firewall rules (Firewall \-\> Rules) to ensure traffic is allowed from LAN/DMZ to WAN. Also, check DNS settings on pfSense (System \-\> General Setup).  
         * **VMs not getting IPs:** Verify DHCP server is enabled on the correct pfSense interfaces (Services \-\> DHCP Server). Check the DHCP range to ensure it's valid and not conflicting.  
         * **Inter-segment communication issues:** This is almost always a pfSense firewall rule issue. Review Firewall \-\> Rules for the source and destination interfaces. Remember that rules are processed top-down.

## **3\. Deploying Your Target Environments: Vulnerable Virtual Machines**

With your network infrastructure (pfSense and bridges) in place, you can now populate your segmented networks with the virtual machines for your pentesting practice. For each VM, remember to connect its network interface to the correct virtual bridge (br-lan, br-dmz, or br-isolated) in virt-manager.

### **3.1. Attacker Machine (LAN Network)**

* **Kali Linux**  
  * **Purpose:** Your primary offensive workstation, equipped with a vast array of security tools for all phases of penetration testing.  
  * **Connect to:** br-lan (will receive IP from pfSense's DHCP server)  
  * **Resources:**  
    * **CPU:** 4 Cores  
    * **RAM:** 8 GiB  
    * **Disk:** 100 GiB (This generous allocation ensures ample room for all pre-installed tools, additional installations like PimpMyKali and Anonsurf, updates, and your custom scripts/data.)  
  * **Configuration Steps:**  
    1. **Install Kali Linux:** Download the official Kali Linux ISO and perform a standard installation as a new VM in virt-manager.  
    2. **Network Configuration:** During installation, ensure its network adapter is configured to obtain an IP address automatically (DHCP). It should successfully receive an IP from pfSense's LAN DHCP server (e.g., 10.0.0.x).  
    3. **Update System:** After installation, open a terminal in Kali and update its package lists and upgrade installed packages:  
       sudo apt update && sudo apt full-upgrade \-y

       **Note on Internet Connectivity for Updates:** For Kali Linux to download updates, it will require outbound internet access. Ensure your pfSense firewall rules allow traffic from the LAN network (where Kali resides) to the WAN (internet). If updates fail, check ping 8.8.8.8 and dig google.com from Kali to diagnose network/DNS issues.  
    4. **Install PimpMyKali/Anonsurf (Optional but Recommended):** These tools enhance Kali's functionality and privacy. **Refer to their respective GitHub pages or official documentation for the most up-to-date installation instructions, as these can change frequently.**

### **3.2. Active Directory Security Targets (LAN Network)**

This section focuses on building a realistic Active Directory environment for practicing domain-level attacks, lateral movement, and privilege escalation within Windows infrastructure.

* **Windows Server 2019 (Domain Controller)**  
  * **Purpose:** The central component of your Active Directory lab, acting as the Domain Controller (DC), DNS server, and DHCP server for the LAN.  
  * **Connect to:** br-lan  
  * **Resources:**  
    * **CPU:** 4 Cores  
    * **RAM:** 8 GiB  
    * **Disk:** 100 GiB (Provides sufficient space for the OS, AD database, logs, and future growth.)  
  * **Configuration Steps:**  
    1. **Install Windows Server 2019:** Download the Windows Server 2019 ISO (evaluation versions are available from Microsoft) and perform a standard installation (e.g., Datacenter with Desktop Experience) as a new VM in virt-manager.  
    2. **Install VirtIO Drivers (Crucial for Windows VMs):** These drivers are essential for optimal performance and proper device recognition (network adapter, disk controller, ballooning) within the KVM environment.  
       * **Download virtio-win.iso**: On your Arch Linux host, download the latest virtio-win.iso from the official Fedora KVM Virtio Drivers page: [https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso).  
       * **Attach ISO to VM**: In virt-manager, with the Windows Server VM powered off, open its details. Click "Add Hardware," select "Storage," choose "CDROM device," and browse to the downloaded virtio-win.iso file.  
       * **Install Drivers in VM**: Power on the Windows Server VM.  
         * Open File Explorer and navigate to the attached virtio-win CD drive.  
         * Run the virtio-win-gt-x64.msi installer for 64-bit Windows. Follow the prompts to install all VirtIO drivers.  
         * **Alternative (Manual Device Manager):** If devices are not recognized, open Device Manager (right-click Start \-\> Device Manager), right-click on any unrecognized devices (e.g., "Ethernet Controller," "SCSI Controller"), select "Update driver," and point it to the virtio-win CD drive.  
         * **Reboot the VM** after installation to ensure all drivers are loaded correctly.  
         * **Troubleshooting VirtIO:** If Windows doesn't recognize the network adapter or disk, ensure the VirtIO ISO is properly attached and that you've run the installer. Sometimes, Windows Update might install generic drivers; ensure the VirtIO ones are specifically used.  
    3. **Initial Network Configuration (Static IP):**  
       * Open Network and Sharing Center \-\> Change adapter settings.  
       * Right-click your Ethernet adapter \-\> Properties \-\> Internet Protocol Version 4 (TCP/IPv4) \-\> Properties.  
       * Select "Use the following IP address" and assign a **static IP address** within your br-lan subnet. **Crucially, choose an IP address *outside* the DHCP range you configured on pfSense** (e.g., 10.0.0.10).  
       * Set the **Subnet mask** to 255.255.255.0.  
       * Set the **Default gateway** to your pfSense LAN interface IP (10.0.0.1).  
       * For **Preferred DNS server**, initially point it to the pfSense LAN IP (10.0.0.1). **This will be changed to 127.0.0.1 later, after AD DS and DNS roles are installed.**  
    4. **Install Active Directory Domain Services (AD DS), DNS, and DHCP Roles:**  
       * Open Server Manager.  
       * Click "Manage" \-\> "Add Roles and Features."  
       * Follow the wizard: Select "Role-based or feature-based installation."  
       * Select your server.  
       * Under "Server Roles," select:  
         * **Active Directory Domain Services** (AD DS)  
         * **DNS Server**  
         * **DHCP Server**  
       * Continue through the wizard, installing any required features.  
    5. **Promote to Domain Controller:**  
       * After the roles are installed, a yellow notification flag will appear in Server Manager. Click it and select "Promote this server to a domain controller."  
       * Choose "Add a new forest" and provide a root domain name (e.g., pentestlab.local or umbrellacorp.local).  
       * Set a strong Directory Services Restore Mode (DSRM) password (keep it secure\!).  
       * Follow the remaining prompts. The server will reboot to complete the promotion process.  
    6. **Verify AD DS, DNS, DHCP and Update DNS:**  
       * After reboot, log in to the server. Open Server Manager and ensure that AD DS, DNS, and DHCP services are running and healthy.  
       * **Crucial DNS Change**: Go back to your network adapter settings (as in step 3\) and change the **Preferred DNS server** to 127.0.0.1 (localhost). This server is now your authoritative DNS server for the domain.  
       * Verify the DHCP scope is active and correctly configured to hand out IP addresses to other VMs on the br-lan network, with the DC's IP as the primary DNS server.  
       * **Troubleshooting AD/DNS/DHCP:** If other VMs can't join the domain or resolve domain names, check:  
         * DNS settings on the client VMs (must point to the DC's IP).  
         * Time synchronization between DC and clients (can cause authentication issues).  
         * DC's firewall (ensure AD ports are open, though usually handled by role installation).  
         * DHCP scope on pfSense (ensure it's not conflicting with the DC's DHCP if you enabled both, or disable pfSense's DHCP for LAN if the DC will handle it).  
    7. **Create Users and Groups (Optional but Recommended for Realism):**  
       * Use "Active Directory Users and Computers" to create various Organizational Units (OUs), users (e.g., "john.doe," "jane.smith," "svc\_web," "admin\_user"), and groups (e.g., "Domain Admins," "IT Staff," "HR Users"). This provides a more realistic target environment for practicing privilege escalation and lateral movement.  
* **Windows 10 Clients (x2 Recommended)**  
  * **Purpose:** Simulate end-user workstations. These are crucial for practicing client-side attacks, local privilege escalation, and lateral movement techniques within the domain.  
  * **Connect to:** br-lan (will receive IP from pfSense's DHCP server, but DNS will point to the DC)  
  * **Resources (each):**  
    * **CPU:** 2 Cores  
    * **RAM:** 6 GiB  
    * **Disk:** 80 GiB  
  * **Configuration Steps:**  
    1. **Install Windows 10:** Download the Windows 10 ISO (evaluation versions are available from Microsoft) and perform a standard installation as a new VM in virt-manager.  
    2. **Install VirtIO Drivers:** Follow the same steps as for the Windows Server (attach virtio-win.iso and install drivers). **Reboot** after installation.  
    3. **Initial Network Configuration (DHCP with Custom DNS):**  
       * Open Network and Sharing Center \-\> Change adapter settings \-\> Right-click your Ethernet adapter \-\> Properties \-\> Internet Protocol Version 4 (TCP/IPv4) \-\> Properties.  
       * Select "Obtain an IP address automatically" (since pfSense's DHCP will assign it).  
       * Select "Use the following DNS server addresses."  
       * Set **Preferred DNS server** to the **static IP address of your Windows Server 2019 Domain Controller** (e.g., 10.0.0.10). This is absolutely critical for successful domain joining.  
       * Set **Alternate DNS server** to your pfSense LAN IP (e.g., 10.0.0.1) as a fallback, or leave it blank if you want to strictly rely on the DC's DNS.  
    4. **Join the Domain:**  
       * Right-click "This PC" \-\> Properties \-\> Click "Change settings" (under "Computer name, domain, and workgroup settings") \-\> Click "Change..."  
       * Select the "Domain" radio button and enter the domain name you created on your DC (e.g., pentestlab.local).  
       * You will be prompted for credentials. Enter the domain administrator username and password (e.g., Administrator and its password from the DC).  
       * The computer will join the domain and require a reboot.  
       * **Troubleshooting Domain Join:** Common issues include incorrect DNS settings (not pointing to the DC), time skew between client and DC, or firewall blocking on the DC. Ensure ping \<DC\_IP\> and ping \<domain\_name\> work from the client.  
    5. **Verify Domain Join:** After reboot, log in using a domain user account (e.g., pentestlab\\john.doe or john.doe@pentestlab.local).  
    6. **Create Local Accounts (Optional for Local PE Practice):** While primarily for domain practice, you can also create local user accounts (standard and administrator) on these Windows 10 VMs to practice local privilege escalation techniques that don't involve Active Directory.

### **3.3. Web Application Security Targets (DMZ Network)**

This section details the setup for intentionally vulnerable web applications, ideal for practicing OWASP Top 10 vulnerabilities. Deploying them on a Docker Host VM within the DMZ offers efficient resource utilization and easy management.

* **Docker Host VM (e.g., Ubuntu Server)**  
  * **Purpose:** A lightweight Linux VM to host multiple vulnerable web applications as Docker containers. This centralizes web app management and is more resource-efficient than running a full VM for each web app.  
  * **Connect to:** br-dmz (will receive IP from pfSense's DHCP server)  
  * **Resources:**  
    * **CPU:** 2 Cores  
    * **RAM:** 4 GiB  
    * **Disk:** 50 GiB (Sufficient for Ubuntu Server, Docker engine, and multiple container images.)  
  * **Configuration Steps:**  
    1. **Install Ubuntu Server VM:** Use virt-manager to create a new VM. Select a minimal Ubuntu Server (LTS recommended) ISO. Allocate the specified resources. Connect its network interface to br-dmz. Perform a standard installation, ensuring the SSH server is installed for convenient remote management.  
    2. **Update System:** After installation, log in via SSH or directly on the console. Update package lists and upgrade installed packages:  
       sudo apt update && sudo apt upgrade \-y

       **Note on Internet Connectivity for Updates:** For this Ubuntu Server VM to download updates, it will require outbound internet access. Ensure your pfSense firewall rules allow traffic from the DMZ network (where this VM resides) to the WAN (internet).  
    3. **Install Docker and Docker Compose:**  
       * Install Docker Engine:  
         sudo apt install docker.io \-y  
         sudo systemctl enable \--now docker  
         sudo usermod \-aG docker $(whoami) \# Add your user to the docker group for non-sudo docker commands

         *(If you are logged in directly on the VM, log out and log back in to apply group changes for your user, or reconnect your SSH session.)*  
       * Install Docker Compose (often included with docker.io or as a separate package docker-compose):  
         sudo apt install docker-compose \-y  
         \# Note: Newer Docker versions might prefer the 'docker compose' plugin instead of the 'docker-compose' binary.  
         \# You can check its version with 'docker compose version' after Docker installation.

       * **Troubleshooting Docker:** If docker commands require sudo even after adding your user to the docker group and re-logging, try sudo systemctl restart docker. If containers fail to start, check docker logs \<container\_name\_or\_id\>.  
    4. **Deploy OWASP Juice Shop:**  
       * Create a directory for Juice Shop:  
         mkdir \~/juice-shop && cd \~/juice-shop

       * Create a docker-compose.yml file in this directory:  
         version: '3'  
         services:  
           juice-shop:  
             image: bkimminich/juice-shop  
             ports:  
               \- "3000:3000" \# Host\_Port:Container\_Port \- Access via http://\<DockerHostIP\>:3000  
             restart: always

       * Start Juice Shop:  
         docker compose up \-d

       * **Verify:** Access http://\<DockerHostIP\>:3000 from your Kali Linux VM. (Replace \<DockerHostIP\> with the actual IP address of your Docker Host VM in the DMZ). **For detailed usage and challenges, refer to the [OWASP Juice Shop official documentation](https://pwning.owasp-juice.shop/part2/setup.html) or its GitHub repository.**  
    5. **Deploy DVWA (Damn Vulnerable Web Application):**  
       * Create a directory for DVWA:  
         mkdir \~/dvwa && cd \~/dvwa

       * Create a docker-compose.yml file in this directory:  
         version: '3.8'  
         services:  
           db:  
             image: mysql:5.7  
             environment:  
               MYSQL\_DATABASE: dvwa  
               MYSQL\_USER: dvwa  
               MYSQL\_PASSWORD: password  
               MYSQL\_ROOT\_PASSWORD: root  
             volumes:  
               \- db\_data:/var/lib/mysql  
             restart: always  
           web:  
             build: .  
             ports:  
               \- "80:80" \# Host\_Port:Container\_Port \- Access via http://\<DockerHostIP\>  
             depends\_on:  
               \- db  
             volumes:  
               \- ./config.inc.php:/var/www/html/config/config.inc.php  
             restart: always  
         volumes:  
           db\_data:

       * Create a Dockerfile in the same directory (\~/dvwa):  
         FROM php:7.4-apache

         RUN apt-get update && apt-get install \-y \\  
             libapache2-mod-php \\  
             php-mysql \\  
             php-gd \\  
             php-xml \\  
             php-mbstring \\  
             unzip \\  
             git \\  
             && rm \-rf /var/lib/apt/lists/\*

         \# Install DVWA  
         RUN git clone https://github.com/ethicalhack3r/DVWA.git /var/www/html

         \# Configure Apache  
         RUN a2enmod rewrite  
         COPY apache-dvwa.conf /etc/apache2/sites-available/000-default.conf  
         RUN chown \-R www-data:www-data /var/www/html  
         RUN chmod \-R 777 /var/www/html/hackable/uploads  
         RUN chmod \-R 777 /var/www/html/external/phpids/0.6/lib/IDS/tmp

         EXPOSE 80

       * Create apache-dvwa.conf in the same directory (\~/dvwa):  
         \<VirtualHost \*:80\>  
             DocumentRoot /var/www/html  
             \<Directory /var/www/html\>  
                 Options Indexes FollowSymLinks  
                 AllowOverride All  
                 Require all granted  
             \</Directory\>  
             ErrorLog ${APACHE\_LOG\_DIR}/error.log  
             CustomLog ${APACHE\_LOG\_DIR}/access.log combined  
         \</VirtualHost\>

       * Create config.inc.php in the same directory (\~/dvwa):  
         \<?php

         // Database variables  
         $DBMS \= 'MySQL';  
         $DB\_SERVER \= 'db'; \# Changed from '127.0.0.1' to 'db' (service name in docker-compose)  
         $DB\_DATABASE \= 'dvwa';  
         $DB\_USER \= 'dvwa';  
         $DB\_PASSWORD \= 'password';

         // ReCAPTCHA settings (leave blank for lab, or obtain keys if desired)  
         $recaptcha\_public\_key \= '';  
         $recaptcha\_private\_key \= '';

         ?\>

       * Start DVWA:  
         docker compose up \-d \--build

       * **Verify:** Access http://\<DockerHostIP\> from your Kali Linux VM. Navigate to setup.php and click "Create / Reset Database." Default credentials: admin/password. **For detailed usage and challenges, refer to the [DVWA GitHub repository](https://github.com/ethicalhack3r/DVWA).**  
    6. **Deploy WebGoat:**  
       * Create a directory for WebGoat:  
         mkdir \~/webgoat && cd \~/webgoat

       * Create a docker-compose.yml file in this directory:  
         version: '3'  
         services:  
           webgoat:  
             image: webgoat/webgoat-8.0  
             ports:  
               \- "8080:8080" \# Host\_Port:Container\_Port \- Access via http://\<DockerHostIP\>:8080/WebGoat  
             restart: always

       * Start WebGoat:  
         docker compose up \-d

       * **Verify:** Access http://\<DockerHostIP\>:8080/WebGoat from your Kali Linux VM. **For detailed usage and lessons, refer to the [OWASP WebGoat official documentation](https://github.com/WebGoat/WebGoat).**

### **3.4. Network Security & Privilege Escalation Targets (LAN Network)**

This section details the setup of intentionally vulnerable Linux and Windows systems for practicing network reconnaissance, service exploitation, and privilege escalation.

* **Metasploitable 3 (Windows and Ubuntu Builds \- Highly Recommended)**  
  * **Purpose:** Offers a more realistic and complex intentionally vulnerable environment using both Windows and Ubuntu operating systems. This diversity is crucial for practicing a wider range of network security, privilege escalation, and post-exploitation techniques across different platforms, mirroring real-world mixed environments.  
  * **Connect to:** br-lan (both builds will receive IPs from pfSense's DHCP server)  
  * **Resources (per build):**  
    * **CPU:** 4 Cores  
    * **RAM:** 6 GiB  
    * **Disk:** 80 GiB  
  * **Configuration Steps:**  
    1. **Prerequisites on Arch Linux Host:**  
       * **Vagrant:** Install Vagrant for automated VM provisioning:  
         sudo pacman \-S vagrant

       * **Vagrant Libvirt Plugin:** Install the KVM/Qemu provider for Vagrant:  
         vagrant plugin install vagrant-libvirt

       * **Packer:** Metasploitable 3 uses Packer to build the base images. Install it:  
         sudo pacman \-S packer

       * **Git:** To clone the Metasploitable 3 repository:  
         sudo pacman \-S git

       * **Troubleshooting Prerequisites:** If any of these tools fail to install, check your Arch Linux package repositories and internet connectivity. Ensure your \~/.bashrc or \~/.zshrc has export PATH="$PATH:$HOME/.local/bin" if you install Python packages to user-local.  
    2. **Download Metasploitable 3 Project:**  
       * Clone the official Metasploitable 3 repository:  
         git clone https://github.com/rapid7/metasploitable3.git  
         cd metasploitable3

    3. **Build Both Metasploitable 3 VMs:**  
       * **For Windows 2008 R2:** This provides a vulnerable Windows server, excellent for Windows-specific exploitation.  
         vagrant up win2k8 \--provider=libvirt

         *(**Note:** This process is lengthy. It will download the Windows ISO, build the VM, install updates, and configure vulnerabilities. This can take several hours depending on your internet speed and CPU. Be patient\!)*  
         * Default Windows login: vagrant/vagrant.  
       * **For Ubuntu (Linux):** This provides a vulnerable Linux server, complementing other Linux targets and offering different exploitation paths.  
         vagrant up ubuntu \--provider=libvirt

         * Default Ubuntu login: vagrant/vagrant.  
       * **Important Note:** Always specify \--provider=libvirt to ensure Vagrant uses KVM/Qemu, not VirtualBox. **For detailed build instructions and post-setup information, refer to the [Metasploitable 3 GitHub repository](https://github.com/rapid7/metasploitable3).**  
       * **Troubleshooting Vagrant/Packer Builds:**  
         * **Build hangs or fails:** This is often due to insufficient host resources (RAM/CPU), network issues (internet access for downloads), or firewall blocking on the host. Ensure pfSense is allowing outbound traffic from your host (if Vagrant/Packer tries to download through it, though typically it uses the host's direct internet).  
         * **"Provider 'libvirt' not found":** Ensure vagrant-libvirt plugin is installed and your user has libvirt group permissions.  
         * **"No space left on device":** Check your host's disk space. Vagrant builds can consume a lot of temporary space.  
    4. **Verify Metasploitable 3 VM Creation:**  
       * Once vagrant up completes, list the running VMs:  
         virsh list \--all

         You should see both metasploitable3\_win2k8\_default and metasploitable3\_ubuntu\_default listed.  
    5. **Connect Metasploitable 3 VMs to br-lan:**  
       * By default, Vagrant creates its own isolated libvirt network. You need to move both Metasploitable 3 VMs to your br-lan network.  
       * Open virt-manager.  
       * For **each** Metasploitable 3 VM (e.g., metasploitable3\_win2k8\_default and metasploitable3\_ubuntu\_default):  
         * Go to its "Virtual Hardware Details" (click the i icon).  
         * Select the "Network interface" (it will likely be connected to a Vagrant-created network).  
         * Change its "Network source" to "Bridge device" and select br-lan from the dropdown.  
         * **Reboot the VM** to apply network changes. They should now get IPs from pfSense's DHCP server on the LAN network.  
       * **Troubleshooting Network Reconnection:** If the VMs don't get an IP after moving to br-lan, ensure pfSense's DHCP server is active on the LAN interface and that the VM's network adapter is set to DHCP.

### **3.5. MAL Network (Optional but Highly Recommended for Malware Analysis & Exploit Dev)**

This network segment is designed for highly sensitive tasks, ensuring no accidental external connectivity, making it ideal for analyzing potentially malicious code in a controlled environment.

* **Dedicated Linux VM (e.g., for REMnux in Docker)**  
  * **Purpose:** For Linux-based malware analysis, reverse engineering, and exploit development using the specialized tools available in REMnux.  
  * **Connect to:** br-isolated (will receive IP from pfSense's DHCP server on the MAL network)  
  * **Resources:**  
    * **CPU:** 2 Cores  
    * **RAM:** 4 GiB  
    * **Disk:** 120 GiB (Sufficient for the base Ubuntu Server OS, Docker engine, the REMnux Docker image, and space for malware samples and analysis artifacts.)  
  * **Configuration Steps:**  
    1. **Install Ubuntu Server VM:** Similar to the Docker Host VM for web apps, create a new Ubuntu Server VM with the specified resources.  
    2. **Connect to:** br-isolated.  
    3. **Install Docker and Docker Compose:** Follow the same Docker installation steps as for the Web App Docker Host VM (Section 3.3).  
    4. **Deploy REMnux Docker Container:**  
       * Pull the REMnux Docker image:  
         sudo docker pull remnux/remnux-full

       * Run the REMnux container. This command maps the current directory on your host as /home/remnux/data inside the container for easy file sharing. It also maps common ports for analysis tools that might be used within REMnux.  
         sudo docker run \--rm \-it \-p 8000:8000 \-p 8080:8080 \-p 9000:9000 \-v $(pwd):/home/remnux/data remnux/remnux-full

       * You will get a shell inside the REMnux container. You can now use all REMnux tools. To exit the container shell without stopping the container, press Ctrl+P then Ctrl+Q. To stop the container, type exit in the container shell. **For the most current deployment and usage instructions, refer to the [REMnux official documentation](https://remnux.org/docs/install/) or its GitHub repository.**  
       * **Troubleshooting REMnux/Docker:** Ensure you have enough disk space for the REMnux image (it's large). If the container fails to start, check docker logs \<container\_id\> for specific errors.  
* **Dedicated Windows VM with FlareVM**  
  * **Purpose:** For Windows-based malware analysis, reverse engineering, and exploit development. FlareVM is a free, open-source, scriptable Windows-based security distribution designed for reverse engineers, malware analysts, and incident responders. It installs a comprehensive suite of tools (debuggers, disassemblers, static analysis tools, network analysis tools, etc.) on a clean Windows installation.  
  * **Connect to:** br-isolated (will receive IP from pfSense's DHCP server on the MAL network)  
  * **Resources:**  
    * **CPU:** 4 Cores  
    * **RAM:** 8 GiB (FlareVM can be resource-intensive due to the number of tools it installs and runs.)  
    * **Disk:** 200 GiB (Allocate generous disk space to accommodate the base Windows installation, FlareVM's extensive toolset, and sufficient room for malware samples and analysis artifacts.)  
  * **Configuration Steps:**  
    1. **Install Windows 10/Server VM:** Create a new Windows 10 or Windows Server VM (e.g., Windows Server 2019 Datacenter) with the specified resources.  
    2. **Install VirtIO Drivers:** Crucial for performance. Follow the same steps as for other Windows VMs (attach virtio-win.iso and install drivers).  
    3. **Initial Network Configuration:** Configure the network adapter to obtain an IP automatically from pfSense's DHCP on the br-isolated network.  
    4. **Prepare for FlareVM Installation:**  
       * **Disable Windows Defender** (or any other antivirus) and **Windows SmartScreen**. These security features will interfere with the installation of many security tools.  
       * Set PowerShell execution policy to Unrestricted:  
         Set-ExecutionPolicy Unrestricted \-Force

       * Install Git for Windows (if not already present).  
    5. **Install FlareVM:**  
       * Open **PowerShell as Administrator**.  
       * Clone the FlareVM repository:  
         git clone https://github.com/mandiant/flare-vm.git  
         cd flare-vm

       * Run the installation script. This process can take several hours, as it downloads and installs many tools from various sources.  
         .\\install.ps1

         Follow any prompts during the installation. **For the most current installation instructions and prerequisites, refer to the [FlareVM GitHub repository](https://github.com/mandiant/flare-vm).**  
       * **Troubleshooting FlareVM:** Installation can be very long and prone to errors if internet connectivity is unstable or if Windows Defender/SmartScreen are not fully disabled. Ensure your VM has consistent internet access *during installation* (if pfSense rules allow outbound from isolated) and sufficient disk space. If it fails, try re-running the script or checking the PowerShell output for specific errors.

## **4\. Practical Lab Scenarios: Optimizing Your Practice**

To make the most of your lab resources and focus your practice, it's highly recommended to only run the virtual machines relevant to the specific cybersecurity domain you are currently working on. This approach conserves host resources (CPU, RAM) and simplifies troubleshooting by reducing the number of active components.  
Your host system has **32GB RAM**, **32GB Swap**, and **1TB dedicated storage**. The resource totals below represent the *combined* requirements for the VMs running in each scenario.  
Here are recommended VM configurations for different practice scenarios:

---

## Network Security Practice Scenario

**Purpose**: Reconnaissance, enumeration, and exploitation against diverse Linux and Windows targets.
**Active VMs**: `pfSense`, `Kali Linux`, `Metasploitable 3 (Windows)`, `Metasploitable 3 (Ubuntu)`

```
+---------------------+
|     Internet /      |
|   Home Router (DHCP)|
+----------+----------+
           |
           | (Physical NIC / Bridge: br-wan)
           |
+----------+----------+
|   Arch Linux Host   |
|     (KVM/Qemu)      |
+----------+----------+
           |
           | (Virtual Bridge: br-wan)
           |
+----------+----------+
|   pfSense Firewall  |
| (Virtual Machine)   |
|---------------------|
| WAN Interface       |
| (DHCP from Home Rtr)|
| LAN Interface       |
+----------+----------+
           |
           | (Virtual Bridge: br-lan)
           |
+-----------------------------------------------------------------------------------+
|                           LAN Network (10.0.0.0/24)                               |
+-----------------------------------------------------------------------------------+
           |                           |                           |
+----------+----------+  +------------+------------+  +-----------+-----------+
| Kali Linux         |  | Metasploitable 3        |  | Metasploitable 3     |
| (Attacker Machine) |  | (Windows Build)         |  | (Ubuntu Build)       |
| (DHCP from pfSense)|  | (DHCP from pfSense)     |  | (DHCP from pfSense)  |
+--------------------+  +-------------------------+  +----------------------+
```

  * **Total Resources for this Scenario:**  
    * **CPU:** 14 Cores (pfSense: 2, Kali: 4, Metasploitable 3 Win: 4, Metasploitable 3 Ubuntu: 4\)  
    * **RAM:** 24 GiB (pfSense: 4, Kali: 8, Metasploitable 3 Win: 6, Metasploitable 3 Ubuntu: 6\)  
    * **Disk Space:** 310 GiB (pfSense: 50, Kali: 100, Metasploitable 3 Win: 80, Metasploitable 3 Ubuntu: 80\)  
  * **Purpose:** This setup allows you to practice network reconnaissance, vulnerability scanning, and exploitation against diverse Windows and Linux targets within your segmented LAN, all controlled by pfSense.

---

## Web Application Security Practice Scenario

**Purpose**: Practice OWASP Top 10 vulnerabilities on intentionally insecure web apps.
**Active VMs**: `pfSense`, `Kali Linux`, `Docker Host VM` (Juice Shop, DVWA, WebGoat)

```
+---------------------+
|     Internet /      |
|   Home Router (DHCP)|
+----------+----------+
           |
           | (Physical NIC / Bridge: br-wan)
           |
+----------+----------+
|   Arch Linux Host   |
|     (KVM/Qemu)      |
+----------+----------+
           |
           | (Virtual Bridge: br-wan)
           |
+----------+----------+
|   pfSense Firewall  |
| (Virtual Machine)   |
|---------------------|
| WAN Interface       |
| (DHCP from Home Rtr)|
| LAN Interface       |
| DMZ Interface       |
+----------+----------+
           |                         |
           |                         |
(Virtual Bridge: br-lan)   (Virtual Bridge: br-dmz)
           |                         |
+----------+----------+   +----------+-----------+
| Kali Linux         |   | Docker Host VM        |
| (Attacker Machine) |   | (e.g., Ubuntu Server) |
| (DHCP from pfSense)|   | (DHCP from pfSense)   |
+--------------------+   |                       |
                         | +-------------------+ |
                         | | OWASP Juice Shop  | |
                         | | DVWA              | |
                         | | WebGoat           | |
                         | +-------------------+ |
                         +-----------------------+
```

  * **Total Resources for this Scenario:**  
    * **CPU:** 8 Cores (pfSense: 2, Kali: 4, Docker Host: 2\)  
    * **RAM:** 16 GiB (pfSense: 4, Kali: 8, Docker Host: 4\)  
    * **Disk Space:** 200 GiB (pfSense: 50, Kali: 100, Docker Host: 50\)  
  * **Purpose:** This configuration provides your attacker machine and the necessary web application targets in the DMZ, enabling you to focus on web vulnerability exploitation.

---

## Active Directory Security Practice Scenario

**Purpose**: Simulate an enterprise Active Directory environment.
**Active VMs**: `pfSense`, `Kali Linux`, `Windows Server 2019`, `Windows 10 Client(s)`

```
+---------------------+
|     Internet /      |
|   Home Router (DHCP)|
+----------+----------+
           |
           | (Physical NIC / Bridge: br-wan)
           |
+----------+----------+
|   Arch Linux Host   |
|     (KVM/Qemu)      |
+----------+----------+
           |
           | (Virtual Bridge: br-wan)
           |
+----------+----------+
|   pfSense Firewall  |
| (Virtual Machine)   |
|---------------------|
| WAN Interface       |
| (DHCP from Home Rtr)|
| LAN Interface       |
+----------+----------+
           |
           | (Virtual Bridge: br-lan)
           |
+-----------------------------------------------------------------------------------+
|                           LAN Network (10.0.0.0/24)                               |
+-----------------------------------------------------------------------------------+
           |                           |                           |
+----------+----------+  +------------+------------+  +-----------+-----------+
| Kali Linux         |  | Windows Server 2019     |  | Windows 10 Clients   |
| (Attacker Machine) |  | (Domain Controller)     |  | (Domain-joined)      |
| (DHCP from pfSense)|  | (10.0.0.10 - Static IP) |  | (DHCP from pfSense)  |
+--------------------+  +-------------------------+  +----------------------+
```

  * **Total Resources for this Scenario:**  
    * **CPU:** 14 Cores (pfSense: 2, Kali: 4, Windows Server DC: 4, Windows 10 Client x2: 2+2=4)  
    * **RAM:** 32 GiB (pfSense: 4, Kali: 8, Windows Server DC: 8, Windows 10 Client x2: 6+6=12)  
    * **Disk Space:** 410 GiB (pfSense: 50, Kali: 100, Windows Server DC: 100, Windows 10 Client x2: 80+80=160)  
  * **Purpose:** This setup provides a realistic Active Directory environment for practicing domain enumeration, privilege escalation, and lateral movement techniques within a Windows enterprise simulation.

---

## MAL Network / Exploit Development Practice Scenario

**Purpose**: Isolated environment for malware analysis and exploit development.
**Active VMs**: `pfSense` (optional), `REMnux VM`, `FlareVM`

```
+---------------------+
|     Internet /      |
|   Home Router (DHCP)|
+----------+----------+
           |
           | (Physical NIC / Bridge: br-wan)
           |
+----------+----------+
|   Arch Linux Host   |
|     (KVM/Qemu)      |
+----------+----------+
           |
           | (Virtual Bridge: br-wan)
           |
+----------+----------+
|   pfSense Firewall  |
| (Virtual Machine)   |
|---------------------|
| WAN Interface       |
| (DHCP from Home Rtr)|
| MAL Int             |
+----------+----------+
           |
           | (Virtual Bridge: br-isolated)
           |
+-----------------------------------------------------------------------------------+
|                           MAL Network (192.168.100.0/24)                          |
+-----------------------------------------------------------------------------------+
           |                                     |
+----------+----------+             +------------+------------+
| REMnux VM (Linux)  |             | FlareVM (Windows)      |
| (Docker Container  |             | (Malware Analysis)     |
| Host)              |             | (DHCP from pfSense)    |
| (DHCP from pfSense)|             +------------------------+
+--------------------+
```
  * **Total Resources for this Scenario:**  
    * **CPU:** 8 Cores (pfSense: 2, REMnux: 2, FlareVM: 4\)  
    * **RAM:** 16 GiB (pfSense: 4, REMnux: 4, FlareVM: 8\)  
    * **Disk Space:** 370 GiB (pfSense: 50, REMnux: 120, FlareVM: 200\)  
  * **Purpose:** This highly isolated setup provides dedicated Linux and Windows environments for safely analyzing malware, reverse engineering binaries, and developing exploits without risk to your host or other lab segments.

---

## **5\. Key Lab Management Practices**

These practices are essential for maintaining a functional, secure, and efficient cybersecurity lab. Adhering to them will maximize your learning experience and protect your host system.

### **5.1. Snapshot Management (Crucial\!)**

Snapshots are an indispensable feature for any cybersecurity lab. They allow you to quickly save the state of a virtual machine at a particular point in time and revert to it later. This is invaluable for resetting vulnerable targets to a clean, known-good state after an exploit, or if a system becomes unstable during an exercise.

* **How to Take Snapshots:**  
  * In virt-manager, select the VM you wish to snapshot.  
  * Click the "Snapshots" icon (it typically looks like a camera).  
  * Click the "Create" button, give it a descriptive name (e.g., "Clean Install," "Post-AD Setup," "Pre-Exploit-X"), and an optional description.  
* **When to Take Snapshots:**  
  * Immediately after a clean operating system installation.  
  * After installing and configuring major services (e.g., after promoting the Domain Controller, after setting up Docker and web apps).  
  * Before attempting any potentially destructive exploits or significant system changes.  
  * Before major updates to a VM that might break its intended vulnerable state.  
* **Reverting to a Snapshot:** Simply select the desired snapshot from the list and click "Revert." The VM will return to the exact state it was in when the snapshot was taken.

### **5.2. Lab Isolation and Security**

The primary goal of a virtual lab is to provide a safe, isolated environment for practicing offensive security. Strict adherence to isolation principles is paramount to prevent accidental network exposure or compromise of your host machine or home network.

* **Strict Segregation:** The use of isolated virtual networks (br-lan, br-dmz, br-isolated) is the most critical security measure. These networks prevent accidental traffic or exploits from your lab reaching your host machine or other devices on your home/production network.  
* **Firewall Rules:** Regularly review and refine your pfSense firewall rules. Ensure that traffic is only allowed where explicitly needed for a lab exercise. Always apply a "default-deny" principle, meaning only explicitly permitted traffic is allowed.  
* **No Sensitive Data:** **Never** store any sensitive personal or work data on your lab VMs, especially the vulnerable targets. Assume that any data on these VMs could be compromised.  
* **Host Security:** Keep your Arch Linux host system fully patched and secure. This mitigates potential hypervisor-level vulnerabilities (like VM escapes) that could allow a compromised guest VM to affect the host system or other virtual machines.

### **5.3. Resource Management**

With 32GB of physical RAM and 32GB of swap memory, you have a robust system. However, running all VMs simultaneously at their maximum recommended allocations will likely exceed your physical RAM, causing the system to rely heavily on swap. While swap provides a buffer, it is significantly slower than RAM and can impact performance.

* **Prioritize Running VMs:** You will rarely need all VMs running at once. Develop a habit of powering on only the VMs relevant to your current task or exercise.  
  * **Example: Active Directory Practice:** Power on pfSense, Kali, Windows Server DC, and 1-2 Windows 10 Clients.  
  * **Example: Web App Pentesting:** Power on pfSense, Kali, and the Docker Host VM.  
  * **Example: MAL Network:** Power on pfSense (if internet access for samples is needed), and either the REMnux VM or FlareVM.  
* **Monitor Resources:** Use host-level tools like htop (on Arch Linux) or virt-manager's built-in statistics to monitor CPU, RAM, and disk I/O usage. This helps identify bottlenecks and informs your VM management decisions.  
* **Adjust Allocations:** If a specific VM consistently runs out of memory or is too slow for a particular task, consider temporarily allocating more RAM or CPU cores to it, provided other non-essential VMs are off.

### **5.4. Updates**

* **Host OS:** Regularly update your Arch Linux host system (sudo pacman \-Syu). This is crucial for security patches and performance improvements for your hypervisor.  
* **Guest OSes:** Regularly update your Kali Linux, Ubuntu Server, and Windows VMs (especially the Kali attacker machine and any Docker host VMs). For intentionally vulnerable targets (like Metasploitable), you might choose to keep them unpatched for specific exercises, but be aware of the risks if they are exposed to the internet.

By meticulously following this comprehensive guide, you will establish a powerful, flexible, and secure virtual cybersecurity lab. This environment will serve as an excellent platform for hands-on learning, skill development, and honing your expertise in penetration testing and red teaming. Good luck with your cybersecurity journey\!
