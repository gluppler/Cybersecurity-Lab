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

If you want this rendered as a diagram, you can use this inside triple backticks (\`\`\`) for a monospaced block or use code blocks in Markdown viewers like GitHub or VSCode for proper alignment.

---

Would you like me to help you generate a diagram using Mermaid or any other diagram markup language for a cleaner visual?
