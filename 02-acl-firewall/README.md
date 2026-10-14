# Zone-Based ACL Firewall

## Problem
After Project 1, the office network was segmented into VLANs but every VLAN could still reach every other one. A guest could ping the admin PC, staff could reach the management network, and any user could try to log in to the switches and router.

## Solution
Turned router R1 into a zone-based packet filter using named ACLs:
- **STAFF-IN** (extended): staff may use only web, DNS and ping on the server, and reach the Internet. All other internal access is blocked.
- **GUEST-IN** (extended): guests get DHCP, DNS and the Internet only, and no internal networks.
- **SERVERS-IN** (extended): the server may only reply (`established`), so a compromised server cannot start connections to users.
- **MGMT-ACCESS** (standard): applied to the VTY lines of every device so only the Admin VLAN can SSH in.
- Added a server VLAN (40) with web and DNS, and a simulated ISP and Internet web server.

## Security policy

| From → To | Staff | Admin | Guest | Servers | Device mgmt | Internet |
|-----------|-------|-------|-------|---------|-------------|----------|
| **Staff** | ✅ | ❌ | ❌ | Web, DNS, ping | ❌ | ✅ |
| **Admin** | ✅ | ✅ | ✅ | ✅ | ✅ SSH | ✅ |
| **Guest** | ❌ | ❌ | ✅ | DNS only | ❌ | ✅ |
| **Servers** | replies only | replies only | replies only | ✅ | ❌ | ❌ |

## Topology
![topology](topology.png)

```
                   [Internet-Web] 198.51.100.10
                          |
                        [ISP]
                          | 203.0.113.0/30
                        [R1]  router-on-a-stick + ACLs
                          |
                [SW1] ─────────── [SW2]
           Staff, Admin, Server   Staff, Guests
```

## Addressing (new in this project)

| Network | Device | Address |
|---------|--------|---------|
| VLAN 40 SERVERS 192.168.40.0/24 | R1 G0/0.40 / Server | 192.168.40.1 / 192.168.40.10 |
| WAN 203.0.113.0/30 | R1 G0/1 / ISP G0/0 | 203.0.113.1 / 203.0.113.2 |
| Internet 198.51.100.0/24 | ISP G0/1 / Internet-Web | 198.51.100.1 / 198.51.100.10 |

VLANs 10, 20, 30 and 99 are the same as in [Project 1](../01-secure-office/).

## Configurations
- [R1](configs/R1.txt)
- [ISP](configs/ISP.txt)
- [SW1](configs/SW1.txt)
- [SW2](configs/SW2.txt)

Passwords are redacted.

## Test results

| Test | From → To | Expected | Result |
|------|-----------|----------|--------|
| T1 | Guest → Admin (ping) | Blocked | |
| T2 | Guest → Staff (ping) | Blocked | |
| T3 | Guest → www.internet.lab | Allowed | |
| T4 | Staff → Admin (ping) | Blocked | |
| T5 | Staff → www.office.lab | Allowed | |
| T6 | Staff → SSH to SW1 | Blocked | |
| T7 | Admin → SSH to SW1 | Allowed | |
| T8 | Admin → Staff (ping) | Allowed | |
| T9 | Staff → Server (ping) | Allowed | |
| T10 | Guest → Server (ping) | Blocked | |
| T11 | Server → Staff (ping) | Blocked | |
| T12 | Staff → Internet-Web (ping) | Allowed | |

<!-- Fill in the Result column with ✅ / ❌ and link screenshots from the screenshots/ folder -->

## Lessons demonstrated
- **Implicit deny:** removing the DHCP permit from GUEST-IN stopped guests getting an IP address.
- **First match wins:** placing a broad deny above a specific permit blocked web access that should have worked.
- **Direction matters:** applying GUEST-IN outbound instead of inbound broke guest Internet access while filtering nothing useful.
- **DAI and static IPs:** the server VLAN was kept out of Dynamic ARP Inspection, because a static IP has no DHCP snooping binding.

## What I learned
- Writing the security policy table first made the ACLs much easier. Each row of the table became one or two ACL lines.
- Order matters: an ACL stops at the first line that matches, so specific permits have to come before broad denies.
- Every ACL ends with an invisible "deny any". When I removed the DHCP permit from GUEST-IN, guests stopped getting an IP address.
- Direction matters as much as content. Applying an ACL inbound on the VLAN subinterface filters traffic as close to the source as possible.
- The `established` keyword lets the server answer requests without being able to start its own connections, which limits the damage if it is ever compromised.
- A standard ACL on the VTY lines (`access-class MGMT-ACCESS in`) is a simple way to make sure only the Admin VLAN can SSH into the devices.

## Tools
Cisco Packet Tracer 8.x, Cisco 2911 routers, Cisco 2960 switches, Server-PT (HTTP, DNS)
