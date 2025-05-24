Certainly! Here's your network diagram converted into a clean Markdown code block using ASCII art, preserving the structure and formatting for easy reading in Markdown:

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
                                     | MAL Int             |  
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

Here is your complete **Cybersecurity Lab Usage Scenarios** section rewritten in clean, properly formatted **Markdown**, with indentation and code blocks for diagrams:

---

# Cybersecurity Lab Usage Scenarios: Network Diagrams

These diagrams visualize the active components and network flow for each of the practical lab scenarios outlined in the setup guide.

---

## 1. Network Security Practice Scenario

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

---

## 2. Web Application Security Practice Scenario

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

---

## 3. Active Directory Security Practice Scenario

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

---

## 4. MAL Network / Exploit Development Practice Scenario

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

---


