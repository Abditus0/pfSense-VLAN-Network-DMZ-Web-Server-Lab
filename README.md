# Enterprise Network Segmentation Lab

Four VLANs running off a pfSense router. Management for IT, Corporate for employees, Guest for visitors, and a DMZ for a public web server. Each one isolated, each one with its own firewall rules, each one tested from every other VLAN to prove the isolation works.

The whole thing runs in VirtualBox. pfSense (the router) handles routing, DHCP, DNS, and firewalling between VLANs. Each VLAN sits on its own host-only adapter so the segmentation is real, not just logical. The DMZ web server is a hardened Ubuntu box running Nginx with UFW layered on top of pfSense, so there are two firewalls between the internet and anything important.

## Network architecture

| VLAN | Purpose | Subnet | Gateway |
|------|---------|--------|---------|
| 10 | Management / LAN | 192.168.10.0/24 | 192.168.10.1 |
| 20 | Corporate | 192.168.20.0/24 | 192.168.20.1 |
| 30 | Guest | 192.168.30.0/24 | 192.168.30.1 |
| 40 | DMZ | 192.168.40.0/24 | 192.168.40.1 |

Each VLAN has its own DHCP scope (.10 to .250) handed out by pfSense.

<img width="861" height="541" alt="Network diagram" src="https://github.com/user-attachments/assets/bd438639-fe51-4a3b-9ce8-8c4e6a8254e8" />

The endpoints:

- **pfSense** does routing, DHCP, DNS, and firewalling for everything
- **Management-PC** (Windows 11) lives on VLAN 10, used for admin work
- **Corporate-PC** (Windows 11) lives on VLAN 20, simulates an employee
- **Guest-PC** (Windows 11) lives on VLAN 30, simulates a visitor
- **DMZ-WebServer** (Ubuntu 24.04 + Nginx) lives on VLAN 40

## Firewall rules

This is where the segmentation happens. Each VLAN has its own ruleset and the rules follow **least privilege**. If a VLAN doesn't need to talk to something, it can't.

### Management (VLAN 10)

<img width="965" height="353" alt="Management VLAN firewall rules" src="https://github.com/user-attachments/assets/206ed8b5-b941-4fc9-bc03-7025a0e9b4cd" />

Full access to everything. This is the IT admin VLAN, so it can hit any other VLAN, the pfSense web interface, and SSH into the DMZ on port 2222. Nothing else gets to do that.

### Corporate (VLAN 20)

<img width="978" height="496" alt="Corporate VLAN firewall rules" src="https://github.com/user-attachments/assets/e2216f03-6b5f-476e-a1d8-59e7865f1371" />

Employees can hit the internet and the DMZ web server on ports 80 and 443. They can't reach the Management VLAN, can't touch the pfSense admin interface, and can't SSH into the DMZ. If a corporate machine gets compromised, the attacker has email and the public web server. Nothing else.

### Guest (VLAN 30)

<img width="981" height="484" alt="Guest VLAN firewall rules" src="https://github.com/user-attachments/assets/a2725d33-3978-47d8-ab7d-08cd85deb48a" />

Internet only. Can't reach Management, Corporate, or the DMZ. DNS allowed so they can actually resolve names. This is the rule set you want for visitor WiFi or anything you don't trust.

### DMZ (VLAN 40)

<img width="972" height="392" alt="DMZ VLAN firewall rules" src="https://github.com/user-attachments/assets/35d525df-6480-4afb-97c1-3d90e3fdaf34" />

The DMZ can reach the internet for updates but it cannot start connections back into any internal network. This is the part that matters. If someone pops the web server, the firewall stops them from pivoting. They're stuck on a Linux box with no path forward.

## Hardening the DMZ web server

pfSense is doing the segmentation, but the web server gets its own layer of protection too. Two firewalls between the internet and the box.

SSH moved off port 22 to 2222. Root login disabled. Password authentication disabled, key-based auth only. Cuts out almost all of the automated bot traffic that pounds port 22 24/7.

```bash
# /etc/ssh/sshd_config
Port 2222
PermitRootLogin no
PasswordAuthentication no
```

UFW handles host-level firewalling on top of pfSense:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.10.0/24 to any port 2222 proto tcp
sudo ufw allow 80,443/tcp
sudo ufw enable
```

Final state:

```
To              Action      From
--              ------      ----
2222/tcp        ALLOW IN    192.168.10.0/24
80,443/tcp      ALLOW IN    Anywhere
```

SSH only from Management. HTTP and HTTPS open to anything that can reach the box (which, thanks to pfSense, is anyone the rules say can reach it). Everything else dropped.

## Testing the isolation

Rules on paper mean nothing. I tested every VLAN against every other VLAN to prove the segmentation actually held.

From **Management-PC**: web access to DMZ works, SSH on 2222 works, can ping every gateway. Full access as expected.

From **Corporate-PC**: web access to DMZ works on 80 and 443. SSH to DMZ times out (UFW drops it before pfSense even sees it). Pings to Management gateway dropped at the firewall. Internet works.

From **Guest-PC**: cannot reach Management, Corporate, or DMZ. At all. Pings die. Web requests time out. Only the internet works. Exactly what a guest network should do.

From **DMZ-WebServer**: can pull updates from the internet. Cannot initiate any connection back into Management, Corporate, or Guest. A compromised DMZ is a dead end for the attacker.

## Tech stack

- pfSense (FreeBSD-based router and firewall)
- Ubuntu Server 24.04 LTS for the DMZ host
- Nginx for the web server
- UFW for host-level firewalling
- VirtualBox for the lab
- Windows 11 for the client VMs

## Problems I had to solve

**Firewall rule order tripped me up early.** pfSense processes rules top to bottom and stops at the first match. I had a Guest VLAN where pings to internal networks were still going through, and I couldn't figure out why. Turned out my "block internal" rules were below an "allow everything" rule that was matching first. Moved the block rules to the top, problem gone. Rule order is everything in pfSense.

**Ubuntu installer would not finish on host-only.** First time around I tried to install Ubuntu Server with the network adapter already on the DMZ host-only network. The installer needs internet to pull packages and it just hung. Fix was to install with the adapter on NAT first, finish setup, then shut down and switch the adapter to the DMZ host-only network. After the switch, Nginx and UFW were already configured, the box just came up on the new VLAN.

**pfSense lost configuration after a "power off".** I shut down the pfSense VM the wrong way once and the next boot came up with no firewall rules and no interface assignments. Lesson learned. pfSense has to be shut down through console option 6 (Halt system). Anything else risks corrupting the config. After that I never powered it off any other way.

**Got locked out of SSH after hardening.** Classic. Switched the SSH port to 2222, disabled password auth, restarted SSH, then realized I hadn't actually set up my SSH key first. I had to use the VirtualBox console to log in directly and fix it. Now the rule is: get key auth working first, then disable passwords. Not at the same time.

## What I learned

The biggest one is that segmentation is only as good as your testing. You can write the cleanest rules in the world but until you actually try to ping every gateway from every VLAN, you don't know if the rules do what you think they do. I caught two misconfigurations during testing that the rules looked fine for on paper.

Defense in depth is real and not just a buzzword. pfSense alone would have been fine for most things, but having UFW on the DMZ box means even if someone managed to get a rule wrong upstream, the host itself still drops the traffic. Two layers, two chances to catch a mistake.

Other things that came out of this:

- pfSense rule order matters more than the rule itself
- DHCP scopes should leave room at the bottom for static assignments (.1 to .9) so the gateway doesn't accidentally get reassigned
- Moving SSH off port 22 cuts the noise in your logs by something like 95% on day one
- Always test the lockout scenarios before applying changes that could lock you out

## Why I built it

I wanted to build the kind of network you'd actually find in a small office, not a flat home network with everything on the same subnet. Real environments have segmentation. Real environments have a guest network that can't see the file server, a corporate network that can't touch the firewall, and a DMZ that can't pivot back inside if it gets popped.

Reading about segmentation isn't the same as configuring it, breaking it, and fixing it. Now I've designed VLANs, written the firewall rules, hardened the host on top, tested every isolation boundary, and dealt with the gotchas that show up when you actually do this work. That's the difference between knowing what a DMZ is and being able to build one.
