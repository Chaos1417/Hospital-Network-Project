<div align="center">

# Al-Shifa Smart Hospital — Enterprise Network Infrastructure

### A HIPAA-Aligned, Three-Tier Hierarchical Network with VLAN Segmentation, OSPF Routing, Layered Security & QoS

[![Cisco Packet Tracer](https://img.shields.io/badge/Cisco%20Packet%20Tracer-8.x-1BA0D7?logo=cisco&logoColor=white)](https://www.netacad.com/courses/packet-tracer)
[![Networking](https://img.shields.io/badge/Networking-Enterprise%20Architecture-0A66C2?logo=cisco&logoColor=white)](https://img.shields.io/badge/Networking-Enterprise%20Architecture-0A66C2)
[![Security](https://img.shields.io/badge/Security-ACLs%20%7C%20NAT%20%7C%20Port%20Security-red)](https://img.shields.io/badge/Security-ACLs%20%7C%20NAT%20%7C%20Port%20Security-red)
[![Routing](https://img.shields.io/badge/Routing-OSPF%20Area%200-FF6F00)](https://img.shields.io/badge/Routing-OSPF%20Area%200-FF6F00)
[![QoS](https://img.shields.io/badge/QoS-VoIP%20%7C%20CoS%20Trust-4CAF50)](https://img.shields.io/badge/QoS-VoIP%20%7C%20CoS%20Trust-4CAF50)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

</div>

---

## Project Overview

This project redesigns the network of **Al-Shifa Smart Hospital**, a fictional 600-bed tertiary-care facility that previously operated on a flat, unmanaged network where every clinical department, administrative workstation, and internet-facing system shared a single broadcast domain. That architecture exposed sensitive Electronic Health Records (EHRs) and PACS radiology images to trivial interception, introduced broadcast storm risk across hundreds of networked medical devices, and provided no prioritization for life-critical VoIP and telemetry traffic.

The solution implements a **HIPAA-aligned, three-tier hierarchical network topology** within Cisco Packet Tracer 8.x, deploying **12 isolated VLANs** across a VLSM-subnetted `192.168.10.0/24` address space, inter-VLAN routing via SVIs on redundant Core Multilayer Switches, and **OSPF Area 0** dynamic routing connecting the main campus to a remote Branch Clinic over a simulated serial WAN link. Layered security is enforced through Extended ACLs, NAT Overload (PAT), and Layer-2 hardening via Port Security with sticky MAC learning, PortFast, and BPDU Guard. QoS for VoIP is implemented through a dedicated Voice VLAN with MLS QoS CoS trust boundaries.

All features were systematically verified through connectivity, routing, ACL isolation, NAT translation, and port-security violation tests — every test returned expected outcomes.

---

## Network Topology

![Network Topology](./images/topology.png)

The architecture follows the **Cisco Three-Tier Hierarchical Model**:

| Layer | Devices | Function |
|-------|---------|----------|
| **Core** | 2x Catalyst 3560 Multilayer Switches | Inter-VLAN routing via SVIs, OSPF, EtherChannel (LACP) backbone, default gateways for all 12 VLANs |
| **Distribution** | 2960 Switches | Traffic aggregation from access switches, dual-homed uplinks, 802.1Q trunking, STP enforcement |
| **Access** | 2960 Department Switches | End-device connectivity, VLAN assignment, Port Security, PortFast, BPDU Guard |
| **Edge** | Cisco 2911 Router | NAT Overload (PAT), OSPF adjacency, ISP connectivity, BRANCH_SECURITY ACL, serial WAN to Branch Clinic |
| **Branch** | Cisco 2911 Router | Remote clinic connectivity via serial WAN (HDLC), OSPF, VLAN 120 |

Redundancy at the Core layer is achieved through dual Multilayer Switches with an **EtherChannel bundle (LACP mode active)**, providing both link redundancy and doubled inter-switch bandwidth.

---

## Objectives & Scope

| Objective | Description |
|-----------|-------------|
| **VLAN Segmentation** | 12 isolated broadcast domains mapped to hospital departments — clinical, administrative, server, guest, and voice segments |
| **VLSM Addressing** | Variable-length subnetting of `192.168.10.0/24` with right-sized `/25`, `/26`, `/27` subnets and dedicated `/30` transit links |
| **Inter-VLAN Routing** | SVI-based Layer-3 routing on Core Switches (hardware ASIC forwarding), eliminating Router-on-a-Stick bottleneck |
| **Dynamic Routing (OSPF)** | Single Area 0 OSPF across Core Switches, Edge Router, and Branch Router with passive interfaces on SVIs |
| **Redundancy & High Availability** | Dual Core Switches, EtherChannel (LACP) bundle, dual-homed distribution uplinks |
| **Layer-2 Security** | Port Security (sticky MAC, max 1, shutdown violation), PortFast, BPDU Guard on all access ports |
| **Layer-3 Security (ACLs)** | Extended ACLs — `GUEST_ISOLATION` (blocks guest→clinical, permits guest→server) and `BRANCH_SECURITY` (restricts branch to server+admin only) |
| **NAT Overload (PAT)** | All internal RFC1918 addresses hidden behind single public IP `203.0.113.2` |
| **Network Services** | Centralized DHCP with `ip helper-address` relay on all SVIs, internal DNS with split-DNS architecture |
| **QoS for VoIP** | Dedicated Voice VLAN 110 with `mls qos trust cos` boundaries, CoS 5 Expedited Forwarding |
| **Encrypted Management** | SSH-only access to all network devices; Telnet explicitly disabled |

---

## Technologies & Protocols

- **Switching:** 802.1Q VLAN Trunking, SVIs, EtherChannel (LACP), DTP disabled
- **Routing:** OSPF Area 0 (single backbone), Inter-VLAN Routing via SVIs, static default route to ISP
- **Addressing:** VLSM (`/25` `/26` `/27` `/30`), DHCP with `ip helper-address` relay
- **Security:** Extended ACLs (named), NAT Overload (PAT), Port Security (sticky MAC), BPDU Guard, PortFast, SSH v2
- **QoS:** Voice VLAN, MLS QoS CoS trust, CoS 5 marking for VoIP bearer traffic
- **WAN:** Serial HDLC encapsulation, point-to-point `/30` transit links
- **Services:** Centralized DHCP server, DNS server with split-DNS (internal + forward to `8.8.8.8`)
- **Management:** SSH-only device access, syslog alerts for port-security violations

---

## Key Configurations

### 1. Inter-VLAN Routing & DHCP Relay on Core Switch

Each VLAN is assigned an SVI on the Core Switch acting as its default gateway. The `ip helper-address` directive relays DHCP broadcasts from every VLAN to the centralized server at `192.168.10.34` in the Server Farm. OSPF passive interfaces prevent unnecessary Hello packets on end-host segments.

```ios
! Core Switch 1 — SVI & DHCP Relay Configuration
interface Vlan10
 description Management VLAN
 ip address 192.168.10.1 255.255.255.224
 ip helper-address 192.168.10.34
!
interface Vlan30
 description ICU VLAN
 ip address 192.168.10.65 255.255.255.192
 ip helper-address 192.168.10.34
!
interface Vlan50
 description Radiology / PACS VLAN
 ip address 192.168.11.1 255.255.255.192
 ip helper-address 192.168.10.34
!
interface Vlan100
 description Guest Wi-Fi VLAN
 ip address 192.168.13.1 255.255.255.128
 ip helper-address 192.168.10.34
 ip access-group GUEST_ISOLATION in
!
! OSPF — advertise all VLAN + transit networks; suppress Hellos on SVIs
router ospf 1
 network 192.168.10.0 0.0.3.255 area 0
 network 10.1.1.0 0.0.0.3 area 0
 passive-interface Vlan10
 passive-interface Vlan30
 passive-interface Vlan50
 passive-interface Vlan100
```

### 2. Extended ACL — GUEST_ISOLATION

This named Extended ACL is applied **inbound** on the VLAN 100 SVI. It denies Guest Wi-Fi users from reaching clinical VLANs (ICU, Emergency, Radiology/PACS) while permitting access to the Web/DNS server. The implicit `deny any` at the end blocks all other internal traffic.

```ios
! GUEST_ISOLATION ACL — Applied inbound on VLAN 100 SVI
ip access-list extended GUEST_ISOLATION
 deny   ip 192.168.13.0 0.0.0.127 192.168.10.64 0.0.0.63   ! Block Guest → ICU
 deny   ip 192.168.13.0 0.0.0.127 192.168.11.0  0.0.0.63   ! Block Guest → Radiology/PACS
 deny   ip 192.168.13.0 0.0.0.127 192.168.10.128 0.0.0.63  ! Block Guest → Emergency
 permit ip 192.168.13.0 0.0.0.127 192.168.10.32 0.0.0.31   ! Permit Guest → Server Farm (Web/DNS)
 deny   ip 192.168.13.0 0.0.0.127 any                       ! Block all other internal traffic
```

### 3. Port Security, PortFast & BPDU Guard on Access Ports

Every access-layer port enforces sticky MAC learning with a maximum of one secure address. The Guest Wi-Fi AP port is an exception — scaled to 50 MACs to support dozens of concurrent wireless clients. BPDU Guard immediately err-disables any port receiving a BPDU, preventing unauthorized switch insertion.

```ios
! Access Switch — Port Security + PortFast + BPDU Guard (standard access port)
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 30
 switchport voice vlan 110
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 mls qos trust cos
!
! Guest Wi-Fi AP port — scaled maximum for multi-client wireless
interface FastEthernet0/24
 switchport mode access
 switchport access vlan 100
 switchport port-security
 switchport port-security maximum 50
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
```

---

## Verification & Testing

Every implemented feature was validated through a systematic test suite in Cisco Packet Tracer simulation mode.

### Test Results Summary

| Test | Method | Expected Result | Outcome |
|------|--------|-----------------|---------|
| **DHCP Assignment** | `ipconfig` on client PCs across all VLANs | IP from correct VLAN pool | All 12 VLANs received correct leases |
| **Inter-VLAN Routing** | Ping ICU PC → Server Farm | Reply via SVI gateway | Successful — SVI routing confirmed |
| **OSPF Convergence** | `show ip route` on Core Switch & Edge Router | All 12 VLAN + transit routes marked `O` | All OSPF routes present |
| **WAN/Branch Connectivity** | `traceroute` from Branch PC → HQ Web Server | 3-hop path via serial link | Path: Branch → 10.2.2.0/30 → Edge Router |
| **Guest ACL Isolation** | Ping Guest (100) → ICU (30) | Request Timed Out | ACL blocked — clinical segment protected |
| **Guest Web Access** | Ping Guest (100) → Web Server (20) | Successful reply | Permitted entry in ACL working |
| **NAT Translation** | `show ip nat translations` after internet ping | Inside local → `203.0.113.2` (PAT) | PAT multiplexing confirmed |
| **SSH Management** | SSH from Mgmt PC → Core Switch | Encrypted session; Telnet refused | SSH-only access enforced |
| **Port Security Violation** | Connect second PC to occupied access port | Port enters `err-disabled` | Violation detected, port shut down |
| **Branch ACL Security** | Ping Branch (120) → ICU (30) | Blocked by BRANCH_SECURITY | Remote-site clinical access denied |

### Sample Verification Output

**OSPF Route Table on Core Switch:**
```ios
CoreSwitch# show ip route
Codes: O - OSPF

     10.0.0.0/8 is variably subnetted, 6 subnets
O       10.1.1.4/30 [110/2] via 10.1.1.2, 00:05:12, GigabitEthernet0/1
O       10.2.2.0/30 [110/11] via 10.1.1.2, 00:05:12, GigabitEthernet0/1
     192.168.10.0/24 is variably subnetted, 7 subnets
C       192.168.10.0/27 is directly connected, Vlan10
C       192.168.10.32/27 is directly connected, Vlan20
C       192.168.10.64/26 is directly connected, Vlan30
O       192.168.14.0/26 [110/12] via 10.1.1.2, 00:05:12, GigabitEthernet0/1
```

**NAT PAT Translation Table:**
```ios
EdgeRouter# show ip nat translations
Pro  Inside local       Inside global      Outside local     Outside global
icmp 192.168.11.130:512 203.0.113.2:512   8.8.8.8:512       8.8.8.8:512
icmp 192.168.12.5:513   203.0.113.2:513   8.8.8.8:513       8.8.8.8:513
```

---

## VLAN & Addressing Reference

| VLAN ID | VLAN Name | Department / Function | Subnet | Usable Hosts |
|---------|-----------|-----------------------|--------|--------------|
| 10 | `VLAN_MANAGEMENT` | Network Management | `192.168.10.0/27` | 30 |
| 20 | `VLAN_SERVER_FARM` | DHCP, DNS, Web, FTP Servers | `192.168.10.32/27` | 30 |
| 30 | `VLAN_ICU` | Intensive Care Unit | `192.168.10.64/26` | 62 |
| 40 | `VLAN_EMERGENCY` | Emergency Ward | `192.168.10.128/26` | 62 |
| 50 | `VLAN_RADIOLOGY` | Radiology / PACS | `192.168.11.0/26` | 62 |
| 60 | `VLAN_PHARMACY` | Pharmacy & Dispensary | `192.168.11.64/26` | 62 |
| 70 | `VLAN_ADMIN` | Administration & Finance | `192.168.11.128/25` | 126 |
| 80 | `VLAN_OUTPATIENT` | Outpatient Department (OPD) | `192.168.12.0/25` | 126 |
| 90 | `VLAN_DOCTORS` | Doctors Offices & Staff | `192.168.12.128/26` | 62 |
| 100 | `VLAN_GUEST_WIFI` | Patient / Guest Wireless | `192.168.13.0/25` | 126 |
| 110 | `VLAN_VOIP` | VoIP / IP Telephony | `192.168.13.128/26` | 62 |
| 120 | `VLAN_BRANCH` | Branch Clinic (Remote Site) | `192.168.14.0/26` | 62 |

**Transit Links (Proactive Engineering Improvement):**

| Link | Subnet | Purpose |
|------|--------|---------|
| Transit-1 | `10.1.1.0/30` | Core Switch 1 ↔ Edge Router |
| Transit-2 | `10.1.1.4/30` | Core Switch 2 ↔ Edge Router |
| Transit-3 | `10.2.2.0/30` | Edge Router ↔ Branch Router |
| WAN | `203.0.113.0/30` | Edge Router ↔ Simulated ISP |

---

## How to Run the Project

1. **Clone the repository**
   ```bash
   git clone https://github.com/<your-username>/al-shifa-smart-hospital-network.git
   cd al-shifa-smart-hospital-network
   ```

2. **Install Cisco Packet Tracer**
   - Download **Cisco Packet Tracer 8.x** from [Cisco Networking Academy](https://www.netacad.com/courses/packet-tracer)
   - Install and launch the application

3. **Open the project file**
   - Navigate to the cloned repository directory
   - Open `Al-Shifa_Smart_Hospital.pkt` in Cisco Packet Tracer

4. **Verify the network**
   - Enter **Simulation Mode** to observe packet flows
   - Test DHCP: Click any PC → Desktop → `ipconfig` → verify IP assignment from the correct VLAN pool
   - Test Inter-VLAN Routing: Ping from an ICU PC (`192.168.10.66`) to the Web Server (`192.168.10.36`)
   - Test OSPF: On Core Switch CLI → `show ip route` → verify `O`-marked routes for all VLANs
   - Test ACL Isolation: Ping from a Guest PC (`192.168.13.x`) to an ICU PC → should return *Request Timed Out*
   - Test NAT: Ping `8.8.8.8` from any internal PC, then run `show ip nat translations` on the Edge Router
   - Test Port Security: Connect a second PC to an occupied access port → port should enter `err-disabled` state

5. **Explore configurations**
   - Click any switch or router → CLI tab → review running configuration
   - Key commands: `show running-config`, `show ip route`, `show vlan brief`, `show access-lists`, `show ip nat translations`

---

## Challenges Encountered

| Challenge | Root Cause | Resolution |
|-----------|------------|------------|
| **BPDU Guard on Trunk Ports** | BPDU Guard accidentally applied to inter-switch trunk ports, err-disabling them since trunks exchange BPDUs | Removed BPDU Guard from all trunk/uplink interfaces; restricted to edge access ports only |
| **VLAN Trunk Negotiation Failure** | Trunk ports left in `dynamic auto` mode; DTP defaulted to access mode | Statically configured all inter-switch links with `switchport mode trunk` and disabled DTP |
| **OSPF Adjacency on Transit Links** | Transit `/30` networks omitted from OSPF `network` statements due to different address space (`10.x` vs `192.168.x`) | Added explicit `network` statements for `10.1.1.0/30`, `10.1.1.4/30`, and `10.2.2.0/30` |

---

## Future Enhancements

- **Dedicated NGFW & IDS/IPS** — Cisco Firepower or pfSense for stateful inspection, replacing ACL-only filtering
- **IPv6 Dual-Stack** — Support for medical IoT devices and cloud-connected EHR systems
- **SD-WAN** — Replace serial leased line with encrypted, QoS-aware overlay on commodity broadband
- **IPsec VPN** — Encrypt all WAN transit between Branch Clinic and HQ (HIPAA requirement)
- **802.1X NAC** — RADIUS-based authentication (Cisco ISE) replacing sticky MAC learning

---

## References

1. Cisco Systems. (2023). *Cisco three-tier hierarchical network design model.* Cisco Press.
2. Jassim, H. A. (2024). Design and implementation of a secure and scalable healthcare network infrastructure using VLAN segmentation and ACL-based access control. *IEEE Access*, 12, 48321–48335.
3. Lammle, T. (2020). *CCNA Cisco Certified Network Associate study guide* (8th ed.). Sybex/Wiley.
4. U.S. Department of Health & Human Services. (2022). *HIPAA security rule: Technical safeguards for healthcare networks.* HHS.gov.
5. Forouzan, B. A., & Fegan, S. C. (2021). *Data communications and networking* (5th ed.). McGraw-Hill Education.

---

<div align="center">

**Rauf Hassan** · FA-2023/BSCS/093 · Lahore Garrison University · Computer Networks (Lab) · May 2025

</div>
