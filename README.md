# L2/L3 Network Lab - Enterprise Switch & Routing Configuration

## Project Overview

A comprehensive GNS3-based network lab simulating enterprise LAN/WAN connectivity with advanced switching capabilities, inter-VLAN routing, network segmentation, and edge firewall security. This lab demonstrates practical knowledge of L2/L3 network design relevant to data center and enterprise IT infrastructure roles.

## TOPOLOGY

<img width="854" height="569" alt="Screenshot 2025-12-03 102614" src="https://github.com/user-attachments/assets/1b6c6b10-3cef-4219-9bd9-4587f5a84fc3" />

## PFSENSE Firewall

<img width="667" height="368" alt="image" src="https://github.com/user-attachments/assets/28bae822-6fe5-402d-bfe9-905af2494ede" />

#Lab Setup Steps#

## Lab topology

- Router R1 (Cisco c7200 in GNS3)  
- Layer2Switch (GNS3 built‑in L2 switch)  
- PC1, PC2 (VPCS nodes)  
- pfSense firewall (VMware VM)  
- Cloud1 (GNS3 cloud bound to VMware VMnet1 host‑only network)  

IP plan:

- PC LAN: 10.10.1.0/24  
  - R1 Fa0/0: 10.10.1.1/24  
  - PC1: 10.10.1.10/24, GW 10.10.1.1  
  - PC2: 10.10.1.11/24, GW 10.10.1.1  
- R1 ↔ pfSense LAN: 192.168.10.0/30  
  - R1 Fa1/0: 192.168.10.1/30  
  - pfSense LAN (em1): 192.168.10.2/30  
- pfSense WAN: DHCP from VMware NAT (192.168.233.0/24, example 192.168.233.x)  

***

## Phase 1 – GNS3 project and devices

1) Create project  
- Open GNS3 → File → New Project.  
- Name: `l2-l3-utm-lab`.  
- Accept defaults.

2) Add devices to canvas  
- Drag one `c7200` router (R1).  
- Drag one `Layer 2 Switch`.  
- Drag two `VPCS` nodes (PC1 and PC2).  
- Drag one `Cloud` node (Cloud1).

***

## Phase 2 – VMware pfSense setup

1) Create VMware networks  
- In VMware Virtual Network Editor:  
  - VMnet1: set to “Host‑only”, DHCP enabled (default 192.168.199.0/24).  
  - VMnet8: NAT (default).

2) Create pfSense VM  
- New VM → Custom.  
- 2 vCPUs, 2 GB RAM, 10 GB disk.  
- Add 2 network adapters:  
  - Adapter 1: NAT (connected to VMnet8).  
  - Adapter 2: Host‑only (connected to VMnet1).  
- Mount pfSense CE ISO as CD/DVD.  
- Boot VM and run pfSense installer (Auto/UFS is fine).  
- After install, remove ISO from CD/DVD so it boots from disk.

3) pfSense interface/IP configuration (console)  
- At console menu, choose 1) Assign Interfaces:  
  - WAN: em0  
  - LAN: em1  
  - Do not configure VLANs.  
- Choose 2) Set interface(s) IP address:  
  - For LAN (em1):  
    - IP: 192.168.10.2  
    - Subnet: /30  
    - No DHCP server on LAN.  
  - For WAN (em0):  
    - Use DHCP (pfSense will get an IP from VMware NAT, e.g., 192.168.233.x).  

***

## Phase 3 – Connect pfSense to GNS3

1) Bind Cloud1 to VMnet1  
- Right‑click Cloud1 → Configure.  
- Check “Show special Ethernet interfaces”.  
- Click Refresh and select `VMnet1`.  
- Click “Add”, then OK.

2) Wire the topology  
- R1 Fa0/0 → Layer2Switch port 0.  
- Layer2Switch port 1 → PC1.  
- Layer2Switch port 2 → PC2.  
- R1 Fa1/0 → Cloud1 (VMnet1).  

***

## Phase 4 – Router and PC configuration

1) Configure R1 interfaces

On R1 console:

- Enter config mode:  
  - `enable`  
  - `configure terminal`

- LAN interface Fa0/0 (towards PCs):  
  - `interface FastEthernet0/0`  
  - `ip address 10.10.1.1 255.255.255.0`  
  - `no shutdown`  
  - `exit`

- Interface to pfSense Fa1/0:  
  - `interface FastEthernet1/0`  
  - `ip address 192.168.10.1 255.255.255.252`  
  - `no shutdown`  
  - `exit`

- Default route via pfSense:  
  - `ip route 0.0.0.0 0.0.0.0 192.168.10.2`  
  - `end`

2) Configure PCs (VPCS)

On PC1:

- `ip 10.10.1.10 255.255.255.0 10.10.1.1`

On PC2:

- `ip 10.10.1.11 255.255.255.0 10.10.1.1`

3) Basic connectivity tests

- From PC1: `ping 10.10.1.1` (should succeed).  
- From PC2: `ping 10.10.1.10` (should succeed).  
- From R1: `ping 192.168.10.2` (pfSense LAN).

***

## Phase 5 – Firewall and NAT design (pfSense)

Because the pfSense web GUI may not be reachable directly from the host in all environments, the rules are defined conceptually and documented in `firewall-nat-rules.txt`:

- Outbound NAT (PAT) on WAN:  
  - Source: 10.10.1.0/24.  
  - Translate to: WAN address.  

- LAN rules (evaluated top‑down):  
  - Allow 10.10.1.0/24 → any, TCP 80/443 (web).  
  - Allow 10.10.1.0/24 → any, TCP/UDP 53 (DNS).  
  - Block 10.10.1.0/24 → any, TCP 22 (SSH), with logging.  
  - Allow ICMP from 10.10.1.10 (optional management host).  
  - Implicit deny for anything else.  

- WAN rules:  
  - Block all unsolicited inbound traffic (default).  
  - Allow established/related return traffic (stateful inspection).

These rules are stored as plain‑text documentation in the repo to show firewall/UTM understanding even if the exact GUI configuration differs slightly per environment.

***

## Phase 6 – Verification steps

1) Internal routing  
- From PC1: ping PC2 and R1.  
- From R1: ping pfSense LAN IP (192.168.10.2).  

2) End‑to‑end path (conceptual)  
- PC1 → R1 (10.10.1.1) → pfSense LAN (192.168.10.2) → pfSense WAN → internet.  
- Traffic is NATed at pfSense WAN and filtered by the firewall rules above.

***

