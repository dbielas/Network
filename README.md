# Enterprise Multi-Site Hybrid Network Architecture & Engineering Reference

## 1. Executive Summary & Design Scope

This architecture document specifies the design, implementation, and verification of a dual-site enterprise network infrastructure connecting a primary Headquarters (HQ) campus to a remote Branch office over an untrusted public WAN via Generic Routing Encapsulation (GRE). 

The infrastructure enforces:
* Deterministic Layer 2 switching boundaries with Spanning Tree hardening (PortFast, BPDU Guard, Root Guard).
* Segmentation across enterprise user access, voice, server management, and an isolated perimeter Demilitarized Zone (DMZ).
* Hybrid routing with OSPFv2 Area 0 interior routing and localized Direct Internet Access (DIA) via static default routing at the branch.
* Perimeter security via 1:1 Static NAT, dynamic PAT overload with route-exempt extended access control, and directional ACL state enforcement.

---

## 2. Logical Topology & Addressing Plan

### Physical Interface & VLAN Layout

```text
                                  +-------------------+
                                  |    ISP Gateway    |
                                  |   (Public WAN)    |
                                  +---------+---------+
                                            |
                         203.0.113.0/30     |     198.51.100.0/30
                     +----------------------+----------------------+
                     |                                             |
             Gi0/0   v                                             v   Gi0/0
     +-----------------------+                             +-----------------------+
     |      HQ-EDGE-01       |<===========================>|      BR-EDGE-01       |
     +---+---------------+---+   Tunnel0: 192.168.100.0/30 +-----------+-----------+
   Gi0/2 |               | Gi0/1 (10.10.0.1/30)                        | Gi0/1 (10.20.10.1/24)
         v               v                                             v
   +-----------+   +-----------+                                 +-----------+
   |    DMZ    |   | HQ-CORE-01|                                 | BR-ACC-01 |
   |172.16.50.0|   +-----+-----+                                 +-----+-----+
   +-----------+         |                                             |
                    Trunk / EtherChannel                               v
                     +---+---+                                     Branch PCs
                     |       |
                     v       v
               +-----------+-----------+
               | HQ-ACC-01 | HQ-ACC-02 |
               +-----+-----+-----+-----+
                     |           |
                     +-----+-----+
                           v
                       Campus PCs
```

### IP Addressing Schema

| Device | Interface | IP Address | Subnet Mask | Purpose / Mapping |
| :--- | :--- | :--- | :--- | :--- |
| **HQ-EDGE-01** | `GigabitEthernet0/0` | `203.0.113.2` | `255.255.255.252` | Outside WAN Link (Next Hop: `203.0.113.1`) |
| | `GigabitEthernet0/1` | `10.10.0.1` | `255.255.255.252` | L3 Transit Link to HQ-CORE-01 |
| | `GigabitEthernet0/2` | `172.16.50.1` | `255.255.255.0` | Default Gateway for DMZ Segment |
| | `Tunnel0` | `192.168.100.1` | `255.255.255.252` | GRE Tunnel to Branch |
| **HQ-CORE-01** | `GigabitEthernet0/1` | `10.10.0.2` | `255.255.255.252` | L3 Transit Link to HQ-EDGE-01 |
| | `Vlan10` | `10.10.10.1` | `255.255.255.0` | Default Gateway for HQ Data Subnet |
| | `Vlan20` | `10.10.20.1` | `255.255.255.0` | Default Gateway for HQ Voice Subnet |
| | `Vlan99` | `10.10.99.1` | `255.255.255.0` | Default Gateway for HQ Management / Servers |
| **BR-EDGE-01** | `GigabitEthernet0/0` | `198.51.100.2` | `255.255.255.252` | Outside WAN Link (Next Hop: `198.51.100.1`) |
| | `GigabitEthernet0/1` | `10.20.10.1` | `255.255.255.0` | Default Gateway for Branch LAN |
| | `Tunnel0` | `192.168.100.2` | `255.255.255.252` | GRE Tunnel to HQ |
| **HQ-DMZ-SRV-01**| `NIC` | `172.16.50.10` | `255.255.255.0` | Public Web/App Server (Static NAT: `203.0.113.10`) |
| **HQ-SRV-01** | `NIC` | `10.10.99.10` | `255.255.255.0` | Central Management, DNS, Syslog |
| **PC-HQ-01** | `NIC` | `10.10.10.50` | `255.255.255.0` | Internal Corporate Workstation |

---

## 3. Layer 2 Access & Hardening Specifications

* **VLAN & Trunking:** 802.1Q encapsulation with explicit allowed VLAN lists.
* **Link Aggregation:** Multi-chassis LACP/PAgP EtherChannels spanning access-to-core uplinks.
* **Spanning Tree Architecture:**
  * **Mode:** Rapid Spanning Tree Protocol (PVRST+ / Rapid-PVST).
  * **Root Placement:** `HQ-CORE-01` serves as the primary root bridge across all active VLANs (`spanning-tree vlan 10,20,99 priority 4096`).
  * **Root Guard:** Applied on all downstream-facing distribution switchports (`HQ-CORE-01` $\rightarrow$ `HQ-ACC-01/02`) to prevent rogue root takeovers.
  * **Edge Hardening:** Host-facing switchports operate with `spanning-tree portfast` enabled to bypass listening/learning phases. BPDU Guard is globally active to error-disable ports if rogue switches are attached.
* **Access Port Security:** Configured via `switchport port-security` using `mac-address sticky`, a maximum of 2 allowed addresses per port, and a `restrict` violation action.

---

## 4. Layer 3 Routing & WAN Architecture

* **Underlay Routing (Direct Internet Access):**
  * `HQ-EDGE-01`: Static default route pointing to ISP1 (`203.0.113.1`).
  * `BR-EDGE-01`: Static default route pointing to ISP2 (`198.51.100.1`) for split-tunnel local breakout, bypassing the corporate GRE tunnel for general web traffic.
* **Overlay GRE Tunnel:**
  * Point-to-Point GRE connecting `HQ-EDGE-01` and `BR-EDGE-01`.
  * TCP Maximum Segment Size (MSS) clamped to `1436` bytes and MTU set to `1476` to mitigate packet fragmentation across the 24-byte GRE encapsulation overhead.
* **IGP Design (OSPF Area 0):**
  * Dynamic peering across HQ Core, HQ Edge, and over the `Tunnel0` interface.
  * **Passive-Interface Policy:** Hellos suppressed on the edge DMZ segment (`Gi0/2`); explicitly unsuppressed on transit and tunnel links.
  * **Route Injection:** `HQ-EDGE-01` originates a default route (`default-information originate always`) to provide Internet reachability for the Core switch. The Branch router ignores this Type 5 LSA due to its local administrative distance priority.

---

## 5. Perimeter Security, NAT & DMZ Policy

* **Inside/Outside NAT Demarcation:**
  * `HQ-EDGE-01` WAN (`Gi0/0`) marked as `nat outside`.
  * Transit (`Gi0/1`) and DMZ (`Gi0/2`) links marked as `nat inside`.
* **NAT Policies:**
  * **Static 1:1 NAT:** Public IP mapping to the DMZ web server (`ip nat inside source static 172.16.50.10 203.0.113.10`).
  * **Dynamic PAT (Overload):** RFC 1918 outbound translations for internal users. Governed by an extended access control list that explicitly exempts Branch-bound traffic (`10.10.0.0/16` $\rightarrow$ `10.20.0.0/16`) and GRE Protocol 47 from translation.
* **Perimeter Access Control Lists (ACLs):**
  * **Inbound WAN (OUTSIDE_IN):** Permits IP Protocol 47 (GRE) from the Branch WAN IP, established TCP sessions, and inbound HTTP/HTTPS specifically directed to the DMZ host. Explicit deny-all drops unauthorized traffic.
  * **DMZ Containment (DMZ_RESTRICT):** Isolates the DMZ from initiating lateral sessions into the internal enterprise subnets (`10.10.0.0/16`, `10.20.0.0/16`) while permitting outbound Internet access for patches and updates.

---

## 6. Architecture Decision Record (ADR): DHCP Snooping

* **Context:** Evaluation of Layer 2 DHCP Snooping, Option 82 handling, and Trust boundaries across virtualized emulation.
* **Observed Limitation:** Simulation engine failures where valid DHCP Offers received on explicitly designated `trusted` trunk/EtherChannel ports are discarded (`DHCP_SNOOPING_NONZERO_GIADDR` / logic drops).
* **Workaround Implemented:** DHCP Snooping disabled globally across the emulation environment. Static IP assignments and unrestricted L2 forwarding applied to facilitate functional verification of Layer 3 architectures.
* **Production Deployment Requirement:** On physical hardware, configure `no ip dhcp snooping information option` on access switches, apply `ip dhcp snooping trust` directly on Port-Channel logical interfaces, and deploy `allow-untrusted` on Core uplinks to normalize Option 82 processing.

---

## 7. Verification Matrix & Evidence Collection

| Test Domain | Device Under Test | Command / Probe | Target Success State |
| :--- | :--- | :--- | :--- |
| **L2 Switching** | `HQ-CORE-01` | `show spanning-tree root` | Local switch ID matches Root for VLANs 10, 20, 99 |
| **EtherChannel** | `HQ-ACC-01` | `show etherchannel summary` | Bundled ports display flag `(P)` in port-channel `(SU)` |
| **OSPF Adjacency** | `HQ-EDGE-01` | `show ip ospf neighbor` | Full adjacency on Transit (`Gi0/1`) and Tunnel (`Tunnel0`) |
| **Routing Tables** | `BR-EDGE-01` | `show ip route 0.0.0.0` | Active default path via ISP2 (`198.51.100.1`) with AD 1 |
| **GRE Data Plane** | `BR-EDGE-01` | `ping 10.10.10.50 source Gi0/1` | 5/5 ICMP success traversing Tunnel0 without packet loss |
| **Static NAT** | External Host | `curl http://203.0.113.10` | HTTP 200 OK from DMZ server |
| **DMZ Isolation** | `HQ-DMZ-SRV-01` | `ping 10.10.10.50` | 0/5 ICMP success (Blocked by `DMZ_RESTRICT` ACL) |
| **NAT Translations** | `HQ-EDGE-01` | `show ip nat translations` | Static mapping active; dynamic overload sessions active |