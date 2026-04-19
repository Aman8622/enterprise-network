# Enterprise Campus Network — Cisco Packet Tracer

A fully implemented dual-site enterprise network designed and configured in Cisco Packet Tracer. The topology spans a **Headquarters (HQ)** campus and a remote **Branch** office, interconnected via a site-to-site **IPSec VPN tunnel** over the public internet.

---

## Features

| Feature               | Details                                                |
| --------------------- | ------------------------------------------------------ |
| **Routing**           | OSPF Process 15, Area 0, across all Layer 3 devices    |
| **Redundancy**        | HSRP at HQ & Branch distribution layer                 |
| **Link Aggregation**  | LACP EtherChannel between HQ-MLSW1 & MLSW2             |
| **VPN**               | Site-to-site IPSec IKEv1 (3DES / SHA / Group 2)        |
| **Firewall**          | Cisco ASA at both sites with extended ACLs and NAT     |
| **VLAN Segmentation** | LAN, WLAN, Management, Blackhole VLANs                 |
| **DHCP**              | Centralised DHCP servers in DMZ with IP Helper relay   |
| **Wireless**          | WLC + Lightweight APs via CAPWAP at both sites         |
| **DMZ**               | Isolated server farm (DHCP, DNS, Web, FTP, Email, NTP) |
| **SSH Hardening**     | SSHv2-only VTY access restricted to Management VLAN    |

---

## IP Addressing Summary

### Campus Subnets

| Category      | Network         | Gateway      |
| ------------- | --------------- | ------------ |
| WLAN — HQ     | 10.10.0.0/16    | 10.10.0.1    |
| WLAN — Branch | 10.11.0.0/16    | 10.11.0.1    |
| LAN — HQ      | 172.16.0.0/16   | 172.16.0.1   |
| LAN — Branch  | 172.17.0.0/16   | 172.17.0.1   |
| Management    | 192.168.10.0/24 | 192.168.10.1 |
| DMZ           | 10.10.10.0/27   | 10.20.20.1   |

### Infrastructure Links (Point-to-Point /30)

| Link                  | Network          |
| --------------------- | ---------------- |
| HQ-ISP — Internet     | 20.20.20.0/30    |
| Branch-ISP — Internet | 30.30.30.0/30    |
| HQ-FWL — ISP          | 105.100.50.0/30  |
| Branch-FWL — ISP      | 205.200.100.0/30 |
| HQ-FWL — MLSW1        | 10.20.20.32/30   |
| HQ-FWL — MLSW2        | 10.20.20.36/30   |
| Branch-FWL — B-MLSW1  | 10.20.20.40/30   |
| Branch-FWL — B-MLSW2  | 10.20.20.44/30   |

---

## VLAN Design

| VLAN | Name       | Site   | Ports                        |
| ---- | ---------- | ------ | ---------------------------- |
| 10   | Management | HQ     | Ports 15–20 / g0/1-2 (IT-SW) |
| 20   | LAN        | HQ     | Ports 3–14 (access)          |
| 50   | WLAN       | HQ     | Ports 21–24 (access)         |
| 60   | BLAN       | Branch | Ports 3–20 (access)          |
| 90   | BWLAN      | Branch | Ports 21–24 (access)         |
| 199  | Blackhole  | Both   | Unused ports — shutdown      |

> **Trunk ports:** f0/1–2 on all Layer 2 switches carry 802.1Q tagged traffic to the distribution MLSWs.

---

## Security

### Device Hardening (applied to all 20 devices)

- Console authentication with 3-minute exec timeout
- `service password-encryption` on all passwords
- MOTD banner: `#UNAUTHORISED ACCESS IS PROHIBITED#`
- `no ip domain-lookup` to prevent DNS delays on typos
- Local username database (`cisco` / `cisco`)

### SSH Access

```
ip domain-name cisco.com
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
access-list 2 permit 192.168.10.0 255.255.255.0
access-list 2 deny any
line vty 0 15
 transport input ssh
 login local
 access-class 2 in
```

> SSH access is restricted to the **Management VLAN (192.168.10.0/24)** only.

### Firewall ACL (inbound on OUTSIDE interface)

Permitted protocols: ICMP, DHCP (67/68), DNS (53), HTTP (80), SMTP (25), FTP (20/21), CAPWAP (5246/5247), LWAPP (12222/12223).

### Blackhole VLAN

All unused switch ports are assigned to VLAN 199 and shut down:

```
int range g0/1-2
 switchport access vlan 199
 shutdown
```

---

## Routing — OSPF

All Layer 3 devices run **OSPF Process 15, Area 0**.

| Device       | Router-ID | Key Networks Advertised                                      |
| ------------ | --------- | ------------------------------------------------------------ |
| HQ-MLSW1     | 3.1.3.1   | 10.20.20.36/30, 172.16.0.0/16, 10.10.0.0/16, 192.168.10.0/24 |
| B-MLSW1      | 4.1.4.1   | 10.20.20.40/30, 172.17.0.0/16, 10.11.0.0/16                  |
| B-MLSW2      | 5.1.5.1   | 10.20.20.44/30, 172.17.0.0/16, 10.11.0.0/16                  |
| HQ-FWL / ISP | 7.1.7.1   | 105.100.50.0/30, 20.20.20.0/30                               |
| ISP (hub)    | 8.1.8.1   | 30.30.30.0/30, 20.20.20.0/30                                 |
| Branch-FWL   | 9.1.9.1   | 30.30.30.0/30, 205.200.100.0/30                              |

---

## IPSec VPN

Site-to-site tunnel between **HQ-FWL (105.100.50.2)** ↔ **Branch-FWL (205.200.100.2)**.

| Parameter      | Value                    |
| -------------- | ------------------------ |
| IKE Version    | IKEv1                    |
| Encryption     | 3DES                     |
| Integrity      | SHA-1                    |
| Authentication | Pre-shared key (`cisco`) |
| DH Group       | Group 2                  |
| Lifetime       | 86,400 s                 |
| ESP Transform  | `esp-3des esp-sha-hmac`  |

Traffic encrypted: all flows between Branch subnets (`172.17.0.0/16`, `10.11.0.0/16`) and HQ subnets (`172.16.0.0/16`, `10.10.0.0/16`, `192.168.10.0/24`, `10.20.20.0/27`).

---

## DMZ Server Farm

Located at HQ on subnet `10.10.10.0/27`, hosted on DMZ-SW.

| Server       | Service                | Protocol/Port |
| ------------ | ---------------------- | ------------- |
| DHCP-1       | Primary DHCP           | UDP 67/68     |
| DHCP-2       | Secondary DHCP         | UDP 67/68     |
| DNS          | Domain Name Resolution | UDP/TCP 53    |
| Web Server   | HTTP                   | TCP 80        |
| FTP Server   | File Transfer          | TCP 20/21     |
| Email Server | SMTP                   | TCP 25        |
| NTP Server   | Time Sync              | UDP 123       |

DHCP is centralised — clients at both sites obtain addresses via `ip helper-address` relay pointing to `10.20.20.5` and `10.20.20.6`.

---

## Wireless

| Parameter          | HQ                 | Branch             |
| ------------------ | ------------------ | ------------------ |
| VLAN               | 50 (WLAN)          | 90 (BWLAN)         |
| Subnet             | 10.10.0.0/16       | 10.11.0.0/16       |
| Gateway (HSRP VIP) | 10.10.0.1          | 10.11.0.1          |
| AP Switch Ports    | f0/21–24           | f0/21–24           |
| Controller         | HQ-WLC             | Branch-WLC         |
| Protocol           | CAPWAP (5246/5247) | CAPWAP (5246/5247) |

---

## High Availability

### HSRP

Configured on all SVI interfaces of the multilayer switches. The virtual IP (`.1`) is the default gateway for clients. Active/Standby roles are assigned per site.

### LACP EtherChannel

A port-channel bundle links **HQ-MLSW1** and **HQ-MLSW2**, providing aggregated bandwidth and link-level redundancy between the two core switches.

### STP PortFast & BPDU Guard

Enabled on all access-layer ports (`f0/3–24`) on the Branch MLSWs to accelerate endpoint connectivity and block rogue switch connections.

---

## Default Credentials

| Field              | Value   |
| ------------------ | ------- |
| Enable password    | `cisco` |
| Console password   | `cisco` |
| SSH username       | `cisco` |
| SSH password       | `cisco` |
| VPN pre-shared key | `cisco` |

---

## Requirements

- **Cisco Packet Tracer** 8.x or later
- Devices used: Cisco 2960 (Layer 2 switches), Cisco 3650 (Multilayer switches), Cisco ASA 5506-X (Firewalls), Cisco 2911 (Routers), Cisco 2504 WLC, Cisco LAP-PT APs

---
