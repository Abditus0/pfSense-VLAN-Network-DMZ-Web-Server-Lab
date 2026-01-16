# Enterprise Network Segmentation Lab with pfSense

## Project Description

A complete enterprise-style network segmentation lab built using **pfSense**, **VLANs**, and a **hardened DMZ web server**. This project includes: 
-  Network segmentation with 4 isolated VLANs
-  Firewall rule design following least-privilege principle
-  DMZ architecture for public-facing services
-  SSH hardening with UFW firewall rules
-  Inter-VLAN routing and access control
-  Guest network isolation (internet-only access)
-  Management network security (admin-only access)


---

## Technologies Used

- **pfSense** (FreeBSD-based router/firewall)
- **VirtualBox** (virtualization platform)
- **Ubuntu Server 24.04 LTS** (DMZ web server)
- **Nginx** (web server software)
- **UFW (Uncomplicated Firewall)** (host-based firewall on Ubuntu)
- **Windows 11** (client VMs for testing)

---

## Network Architecture

### VLAN Overview

| VLAN | Purpose | Subnet | Gateway | DHCP Range | Security Level |
|------|---------|--------|---------|------------|----------------|
| **VLAN 10** | Management/LAN | 192.168.10.0/24 | 192.168.10.1 | .10-.250 | Highest |
| **VLAN 20** | Corporate | 192.168.20.0/24 | 192.168.20.1 | .10-.250 | High |
| **VLAN 30** | Guest | 192.168.30.0/24 | 192.168.30.1 | .10-.250 | Low (Isolated) |
| **VLAN 40** | DMZ | 192.168.40.0/24 | 192.168.40.1 | .10-.250 | Medium (Isolated) |

### Devices

| Device | VLAN | IP Address | Purpose |
|--------|------|------------|---------|
| pfSense Router | All | Gateway on each VLAN | Router/Firewall/DHCP Server |
| Management-PC | 10 | 192.168.10.x (DHCP) | IT admin workstation |
| Corporate-PC | 20 | 192.168.20.x (DHCP) | Employee workstation |
| Guest-PC | 30 | 192.168.30.x (DHCP) | Visitor device |
| DMZ-WebServer | 40 | 192.168.40.x (DHCP) | Hardened web server (Nginx) |

### Network Diagram

<img width="861" height="541" alt="515118862-eaed407d-ab35-439f-bd04-ca92c73e2e4e" src="https://github.com/user-attachments/assets/bd438639-fe51-4a3b-9ce8-8c4e6a8254e8" />

---

## Firewall Rules Summary

### Management VLAN (VLAN 10)  <img width="965" height="353" alt="515144742-695de78b-01ed-4e92-9556-3913dda07fca" src="https://github.com/user-attachments/assets/206ed8b5-b941-4fc9-bc03-7025a0e9b4cd" />


```
Allow ALL traffic (default pfSense LAN rule)
```
- Full access to all networks and pfSense management interface
- Only accessible by IT administrators
- Can SSH to DMZ server for management

### Corporate VLAN (VLAN 20)  <img width="978" height="496" alt="515144925-4ccc48a3-fd48-4545-a99b-403f7592d372" src="https://github.com/user-attachments/assets/e2216f03-6b5f-476e-a1d8-59e7865f1371" />

```
Block → Guest VLAN (192.168.30.0/24)
Block → Management VLAN (192.168.10.0/24)
Allow → DMZ port 80/443 only (HTTP/HTTPS)
Allow → Everything else
```
- Employees can access internet and company resources
- **Cannot** access firewall management interface
- Can access DMZ web server via HTTP/HTTPS
- **Cannot** SSH to DMZ server

### Guest VLAN (VLAN 30)  <img width="981" height="484" alt="515144952-c326ae05-b9ab-411b-8065-424986e81f41" src="https://github.com/user-attachments/assets/a2725d33-3978-47d8-ab7d-08cd85deb48a" />

```
Block → LAN/Management (192.168.10.0/24)
Block → Corporate (192.168.20.0/24)
Block → DMZ (192.168.40.0/24)
Allow → Port 53 (DNS)
Allow → Internet
```
- Completely isolated from all internal networks
- Internet access only (perfect for visitors/public WiFi)
- Cannot access any internal resources

### DMZ VLAN (VLAN 40)  <img width="972" height="392" alt="515144975-b58e0f3b-db0c-49ad-9e38-68b8c51a773c" src="https://github.com/user-attachments/assets/35d525df-6480-4afb-97c1-3d90e3fdaf34" />

```
Block → LAN/Management (192.168.10.0/24)
Block → Corporate (192.168.20.0/24)
Block → Guest (192.168.30.0/24)
Allow → Internet (for system updates)
```
- SSH accessible ONLY from Management VLAN (port 2222)
- Cannot initiate connections TO internal networks
- If compromised, attacker cannot pivot to corporate resources

---

## Setup Guide

### Prerequisites

- VirtualBox/Hyper-V/VMware installed (I'm using VirtualBox)
- pfSense ISO (latest version)
- Ubuntu Server 24.04 LTS ISO
- Windows 11 ISO (for test VMs)
- Minimum 16GB RAM (32GB recommended)
- 100GB free disk space

### Phase 1: pfSense Router Setup

1. **Create pfSense VM:**
   - 1GB RAM, 20GB disk
   - 5 network adapters:
     - Adapter 1: NAT Network "WAN-Network" (WAN/Internet)
     - Adapter 2: Host-only Adapter #2 (VLAN 10 - Management)
     - Adapter 3: Host-only Adapter #3 (VLAN 20 - Corporate)
     - Adapter 4: Host-only Adapter #4 (VLAN 30 - Guest)
     - Adapter 5: Host-only Adapter #5 (VLAN 40 - DMZ)

2. **Configure VirtualBox Host-only Adapters:**
   - Adapter #2: 192.168.10.254/24 (DHCP disabled)
   - Adapter #3: 192.168.20.254/24 (DHCP disabled)
   - Adapter #4: 192.168.30.254/24 (DHCP disabled)
   - Adapter #5: 192.168.40.254/24 (DHCP disabled)

3. **Install pfSense:**
   - Boot from ISO
   - Select "Install pfSense"
   - Partition scheme: ZFS (Stripe - no redundancy)
   - Wait for installation to complete
   - Reboot (remove ISO after first boot)

4. **Assign interfaces via console:**
```
   WAN: em0 (NAT Network)
   LAN: em1 (Management VLAN)
   OPT1: em2 (Corporate VLAN)
   OPT2: em3 (Guest VLAN)
   OPT3: em4 (DMZ VLAN)
```

5. **Configure IP addresses (via console menu option 2):**
```
   LAN (em1): 192.168.10.1/24 (DHCP: 10-250)
   OPT1 (em2): 192.168.20.1/24 (DHCP: 10-250)
   OPT2 (em3): 192.168.30.1/24 (DHCP: 10-250)
   OPT3 (em4): 192.168.40.1/24 (DHCP: 10-250)
```

6. **Access web interface:**
   - Create Management-PC VM (Windows 11, connected to Host-only Adapter #2)
   - From Management-PC browser: 192.168.10.1
   - Default credentials: admin/admin
   - **Change password immediately**

7. **Complete setup wizard:**
   - Hostname: pfsense
   - Primary DNS: 8.8.8.8
   - Secondary DNS: 8.8.4.4
   - Timezone: (Your timezone)
   - WAN: DHCP
   - LAN: 192.168.10.1/24

8. **Rename interfaces (Interfaces → Assignments):**
   - OPT1 → **CORPORATE**
   - OPT2 → **GUEST**
   - OPT3 → **DMZ**
   - Enable each interface and Save

9. **Create VMs:**
   - Create Corporate-PC VM (Windows 11, connected to Host-only Adapter #3)
   - Create Guest-PC VM (Windows 11, connected to Host-only Adapter #4)

### Phase 2: DMZ Web Server Setup

#### Step 1: Create and Install Ubuntu Server

1. **Create VM:**
   - Name: DMZ-WebServer
   - Type: Linux
   - Version: Ubuntu LTS (64-bit)
   - Memory: 2048 MB (2GB)
   - Disk: 20GB

2. **Network configuration (for installation):**
   - Settings → Network → Adapter 1
   - Attached to: **NAT** (temporarily, for package downloads)

3. **Install Ubuntu Server:**
   - Start VM
   - Select "Install Ubuntu Server"
   - Language: English
   - Keyboard: Your layout
   - Network: Leave on DHCP (NAT will work)
   - Proxy: Leave blank
   - Mirror: Default
   - Storage: Use entire disk
   - **Profile setup:**
     - Your name: admin
     - Server name: dmz-webserver
     - Username: admin
     - Password: (your password)
   - **SSH: Enable OpenSSH server** ✓
   - Featured snaps: Skip/None
   - Wait for installation to complete
   - Reboot

4. **After first boot, log in:**
   - Username: admin
   - Password: (your password)

#### Step 2: Install and Configure Nginx
```bash
# Update package lists
sudo apt update

# Install Nginx
sudo apt install nginx -y

# Verify Nginx is running
sudo systemctl status nginx
# Should show "active (running)"

# Check IP address (still on NAT)
ip addr show
```

#### Step 3: SSH Hardening

**Change SSH port to 2222:**
```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Add at the bottom: 
PermitRootLogin no
PasswordAuthentication no
port 2222
# Save (Ctrl+O, Enter) and exit (Ctrl+X)

# Restart SSH service
sudo systemctl restart sshd
```

**Configure UFW Firewall:**
```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH from Management VLAN only
sudo ufw allow from 192.168.10.0/24 to any port 2222 proto tcp

# Allow HTTP and HTTPS from anywhere
sudo ufw allow 80,443/tcp

# Enable firewall
sudo ufw enable -y

# Verify rules
sudo ufw status verbose
```

**Expected UFW output:**
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW IN    192.168.10.0/24
80,443/tcp                 ALLOW IN    Anywhere
```

#### Step 4: Move to DMZ VLAN

1. **Shut down the VM:**
```bash
   sudo shutdown now
```

2. **Change network adapter:**
   - Right-click VM → Settings → Network
   - Attached to: **Host-only Adapter**
   - Name: **VirtualBox Host-Only Ethernet Adapter #5** (DMZ VLAN)
   - Click OK

3. **Start VM and verify:**
```bash
   # Check IP address
   ip addr show
   # Should now show: 192.168.40.x (from DHCP)

   # Test internet connectivity
   ping -c 4 8.8.8.8

   # Verify Nginx is running
   sudo systemctl status nginx
```

4. **Note the IP address** (e.g., 192.168.40.x) - you'll use this to access the web server.

### Phase 4: Testing & Validation

#### Test 1: Web Server Access

**From Management-PC:**
```
Browser: http://192.168.40.x
```

**From Corporate-PC:**
```
Browser: http://192.168.40.x
```

**From Guest-PC:**
```
Browser: http://192.168.40.x
Result: Should timeout (blocked by firewall)
```

#### Test 2: SSH Access

**From Management-PC (Command Prompt):**
```bash
ssh -p 2222 YourDMZUsername@192.168.40.x
Result: Should connect successfully
```

**From Corporate-PC:**
```bash
ssh -p 2222 YourDMZUsername@192.168.40.x
Result: Connection should timeout (UFW blocking)
```

**Test commands from each VM:**
```cmd
# Test basic connectivity
ping 192.168.10.1    # Management gateway
ping 192.168.20.1    # Corporate gateway
ping 192.168.30.1    # Guest gateway
ping 192.168.40.1    # DMZ gateway
ping 8.8.8.8         # Internet (Google DNS)

# Test web access
curl http://192.168.40.x
# or use web browser

# Test SSH (from Management-PC only)
ssh -p 2222 YourDMZUsername@192.168.40.x
```

---

## Troubleshooting

### Issue: Can't access pfSense web interface
**Symptoms:** Browser times out when accessing https://192.168.10.1

**Solutions:**
- Verify you're on Management VLAN (run `ipconfig` - should show 192.168.10.x)
- Check VM network adapter is connected to Host-only Adapter #2
- Try pinging gateway first: `ping 192.168.10.1`
- Ensure pfSense VM is running

### Issue: Guest can ping internal networks
**Symptoms:** Guest-PC can successfully ping 192.168.10.1 or 192.168.20.1

**Solutions:**
- Check firewall rule order in pfSense (Firewall → Rules → GUEST)
- Block rules must be ABOVE any allow rules
- Click "Apply Changes" after modifying rules

### Issue: No DHCP address on VM
**Symptoms:** `ipconfig` shows "169.254.x.x" (APIPA address) or no IP

**Solutions:**
- Verify VM network adapter is connected to correct Host-only adapter
- Check DHCP is enabled on that interface in pfSense (Interfaces → [Interface] → scroll to DHCP section)
- Windows: `ipconfig /release` then `ipconfig /renew`
- Linux: `sudo dhclient -r` then `sudo dhclient`

### Issue: DMZ web server not accessible
**Symptoms:** Browser times out when accessing http://192.168.40.x

**Solutions:**
- Verify Nginx is running on DMZ server: `sudo systemctl status nginx`
- Check DMZ server IP: `ip addr show` (should be 192.168.40.x)
- Verify UFW allows port 80: `sudo ufw status verbose`
- Try accessing from Management-PC first (fewer firewall rules)
- Check pfSense firewall logs: Status → System Logs → Firewall

### Issue: Can't SSH to DMZ server from Management-PC
**Symptoms:** Connection timeout or "Connection refused"

**Solutions:**
- Verify SSH is listening on port 2222: `sudo ss -tlnp | grep 2222`
- Check UFW rule: `sudo ufw status verbose` (should allow 2222 from 192.168.10.0/24)
- Verify SSH service is running: `sudo systemctl status sshd`
- Use correct syntax: `ssh -p 2222 YourDMZUsername@192.168.40.x`
- Check Management-PC IP is in 192.168.10.0/24 range: `ipconfig`

### Issue: Ubuntu Server installer hangs during installation

**Solutions:**
- Install with network set to NAT (provides faster internet during install)
- After installation, shut down and change to Host-only adapter

### Issue: pfSense loses configuration after reboot
**Symptoms:** IP addresses and firewall rules disappear after shutting down

**Solutions:**
- **ALWAYS** shut down pfSense properly:
  - Console menu → Option 6 (Halt system)
- **NEVER** use "Power off" - this corrupts config file
- If already corrupted: Reinstall and reconfigure from scratch

---

### Networking Concepts
- VLAN design and implementation
- IP addressing and subnetting (/24 networks)
- Inter-VLAN routing configuration
- DHCP server setup and scope management
- DNS configuration (forwarders)
- Network segmentation principles

### Security Implementation
- Firewall rule creation and management (pfSense)
- Host-based firewall configuration (UFW)
- SSH hardening (non-standard port, root login disabled)
- Network segmentation for security
- DMZ architecture and isolation
- Least privilege access control
- Guest network isolation
- Source-based access control (Management-only SSH)

### Systems Administration
- pfSense router/firewall deployment and configuration
- Linux server administration (Ubuntu Server)
- Web server installation and configuration (Nginx)
- Service management (systemctl)
- SSH configuration and hardening
- Firewall rule management (UFW)
- Virtual machine configuration (VirtualBox)
- Network troubleshooting methodology
- Documentation and diagramming

### Problem-Solving
- Troubleshooting network connectivity issues
- Debugging firewall rule conflicts
- Resolving DHCP assignment problems
- Identifying and fixing security misconfigurations
- Systematic testing and validation

---

## Real-World Applications

This lab simulates enterprise network designs found in:

**Small-to-Medium Businesses:**
- Separate employee network from guest WiFi
- Public-facing web servers in DMZ
- Management network for IT staff only
- Corporate resources protected from guests

**Coffee Shops / Hotels / Public Venues:**
- Guest WiFi completely isolated from business operations (POS systems, employee devices)
- Prevents guests from accessing internal resources
- Public access without security risk

**Corporate Offices:**
- Employee network segmented by department/role
- Management network for administrators
- DMZ for public-facing services (website, email gateway, VPN endpoint)
- Compliance with security frameworks (PCI-DSS, HIPAA, etc.)

**Data Centers:**
- Network segmentation for multi-tenant environments
- DMZ for customer-facing services
- Backend networks isolated from public access
- Security compliance and audit requirements

**Managed Service Providers (MSPs):**
- Client network segmentation
- Separate management plane from customer networks
- Secure remote access architecture

---

## Project Notes

### Design Decisions

**Why 4 VLANs?**
- Balances complexity with learning value
- Covers most common enterprise scenarios (Management, Corporate, Guest, DMZ)
- Manageable in a home lab environment

**Why Nginx instead of Apache?**
- Lighter resource usage (important in VM environment)
- Simpler configuration for basic web hosting
- Industry standard for modern web applications
- Better performance for static content

**Why SSH on port 2222 instead of 22?**
- Reduces automated bot attacks (port 22 is heavily targeted)
- Security through obscurity (not a replacement for strong auth, but helps)
- Common practice in production environments

**Why UFW instead of iptables?**
- More user-friendly (easier to understand rules)
- Perfect for learning firewall concepts
- Still uses iptables under the hood
- Production-ready (used on many Ubuntu servers)

## What I Learned

### Technical Skills
- Configured a multi-VLAN network from scratch
- Implemented security segmentation following best practices
- Hardened a Linux server with firewall rules and SSH security
- Deployed and configured pfSense in an enterprise-style topology
- Troubleshot network connectivity issues systematically
- Created network documentation

### Soft Skills
- Technical documentation and diagramming
- Systematic testing and validation methodology
- Attention to detail (IP addressing, firewall rule order)
- Self-directed learning (figuring out VirtualBox limitations)

### Key Takeaways
- **Defense in depth works:** Multiple security layers (pfSense firewall + UFW) provide better protection
- **Rule order matters:** Firewall processes rules top-to-bottom - block rules must come first
- **Testing is crucial:** Always validate each configuration change
- **Documentation saves time:** Clear diagrams and IP schemes prevent mistakes

---

## License

This project documentation is released under the **MIT License** - feel free to use, modify, and share for educational purposes.

