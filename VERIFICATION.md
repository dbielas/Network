## 7. Verification Matrix & Evidence Collection

### 7.1 Control Plane Verification Artifacts

#### Test CP-01: Spanning Tree Root Bridge Placement & Path Hardening
* **Objective:** Validate that `HQ-CORE-01` serves as the deterministic 802.1w root bridge for all enterprise VLANs (10, 20, 99) with non-default bridge priority `4096`.
* **Execution Node:** `HQ-CORE-01`

```text
HQ-CORE-01#show spanning-tree summary 
Switch is in rapid-pvst mode
Root bridge for: DATA_CORP VOICE_GUEST VLAN0099
Extended system ID           is enabled
Portfast Default             is enabled
PortFast BPDU Guard Default  is enabled
Portfast BPDU Filter Default is disabled
Loopguard Default            is disabled
EtherChannel misconfig guard is disabled
UplinkFast                   is disabled
BackboneFast                 is disabled
Configured Pathcost method used is short

Name                   Blocking Listening Learning Forwarding STP Active
---------------------- -------- --------- -------- ---------- ----------
VLAN0001                     6         0        0          2          8
VLAN0010                     6         0        0          2          8
VLAN0020                     6         0        0          2          8
VLAN0099                     6         0        0          2          8

---------------------- -------- --------- -------- ---------- ----------
4 vlans                     24         0        0          8         32               Desg FWD 9         128.28   P2p 
```

---

#### Test CP-02: Multi-Link Aggregation (L2 EtherChannel Integrity)
* **Objective:** Confirm LACP/PAgP bundling status across Core-Access trunks (`Po1` and `Po2`) without orphaned physical members.
* **Execution Nodes:** `HQ-ACC-01` and `HQ-ACC-02`

```text
HQ-ACC-01#show etherchannel summary
Flags:  D - down        P - in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------------------------
1      Po1(SU)           LACP   Gig0/1(P) Gig0/2(P) 

HQ-ACC-02# show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------------------------
2      Po2(SU)           LACP   Gig0/1(P) Gig0/2(P) 
```

---

#### Test CP-03: Multi-Area OSPF Adjacency & Interface State
* **Objective:** Verify `HQ-EDGE-01` establishes `FULL` adjacency simultaneously with `HQ-CORE-01` in Area 0 and `BR-EDGE-01` in Area 1 across `Tunnel0`.
* **Execution Node:** `HQ-EDGE-01`

```text
HQ-EDGE-01# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   FULL/DR        00:00:34    10.255.0.2      GigabitEthernet0/0/1
3.3.3.3           0   FULL/  -        00:00:32    10.254.0.2      Tunnel0

HQ-EDGE-01#show ip ospf interface brief
Interface     PID   Area                     IP Address/Mask          Cost  State  Nbrs F/C
Gig0/0/2        1   0                      172.16.50.1/255.255.255.0   1       DR  0/0
Gig0/0/1        1   0                     10.255.0.1/255.255.255.252   1      BDR  0/0
Tun0            1   1                     10.254.0.1/255.255.255.252   1000 POINT  0/0   1     0               172.16.50.1/24     1     P/I   0/0
```

---

#### Test CP-04: Inter-Area LSA Synthesis & Branch Routing Table
* **Objective:** Verify `BR-EDGE-01` populates Area 0 campus and DMZ subnets as Type 3 Inter-Area (`O IA`) routes over `Tunnel0` while preserving its local default gateway via ISP2.
* **Execution Node:** `BR-EDGE-01`

```text
BR-EDGE-01#show ip route ospf 
     10.0.0.0/8 is variably subnetted, 11 subnets, 3 masks
O IA    10.10.10.0 [110/1002] via 10.254.0.1, 00:02:52, Tunnel0
O IA    10.10.20.0 [110/1002] via 10.254.0.1, 00:02:52, Tunnel0
O IA    10.10.99.0 [110/1002] via 10.254.0.1, 00:02:52, Tunnel0
O IA    10.255.0.0 [110/1001] via 10.254.0.1, 00:25:18, Tunnel0
O IA    10.255.0.4 [110/1001] via 10.254.0.5, 00:31:16, Tunnel1
     172.16.0.0/24 is subnetted, 1 subnets
O IA    172.16.50.0 [110/1001] via 10.254.0.1, 00:36:30, Tunnel0

BR-EDGE-01#show ip route 0.0.0.0 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 198.51.100.1
      Route metric is 0, traffic share count is 1
```
### 7.2 Data Plane & Overlay Verification Artifacts

#### Test DP-01: End-to-End Inter-Site GRE Overlay Path (Branch to HQ Access)
* **Objective:** Validate that enterprise inter-site traffic between `PC1` (Branch VLAN 10) and `PC-HQ-01` (HQ Campus VLAN 10) traverses the logical GRE overlay (`Tunnel0` via `10.254.0.1`) without exposing private packets or RFC 1918 addresses directly to the public ISP underlay.
* **Execution Node:** `PC1` (`10.20.10.50`)
* **Command:** `tracert 10.10.10.50`

```text
C:\>tracert 10.10.10.50

Tracing route to 10.10.10.50 over a maximum of 30 hops: 

  1   0 ms      0 ms      0 ms      10.20.10.1  (BR-EDGE-01 LAN Gateway)
  2   2 ms      0 ms      0 ms      10.254.0.1  (HQ-EDGE-01 Tunnel0 Interface)
  3   0 ms      0 ms      0 ms      10.255.0.2  (HQ-CORE-01 Routed Transit)
  4   0 ms      10 ms     0 ms      10.10.10.50 (PC-HQ-01 Target Host)

Trace complete.
```
* **Analysis:** Hop 2 explicitly hits `10.254.0.1`, proving that traffic is steered across `Tunnel0` rather than the public ISP2 gateway (`198.51.100.1`). Hop 3 confirms Layer 3 transit handing off to `HQ-CORE-01` before final delivery down `Po1` to VLAN 10.

---

#### Test DP-02: Branch Direct Internet Access (DIA) Split-Tunnel Path
* **Objective:** Verify split-tunnel policy routing on `BR-EDGE-01` by ensuring external web and DNS traffic directed to `INET-WEB-01` breaks out directly across `ISP2` without hair-pinning across the corporate GRE tunnel.
* **Execution Node:** `PC1` (`10.20.10.50`)
* **Command:** `tracert 8.8.8.10`

```text
C:\>tracert 8.8.8.8

Tracing route to 8.8.8.8 over a maximum of 30 hops: 

  1   0 ms      0 ms      0 ms      10.20.10.1   (BR-EDGE-01 LAN Gateway)
  2   0 ms      0 ms      10 ms     198.51.100.1 (ISP2 Public Gateway)
  3   *         0 ms      0 ms      8.8.8.8      (INET-WEB-01 Public Host)

Trace complete.
```
* **Analysis:** Hop 2 routes directly through `ISP2` (`198.51.100.1`), validating that the branch static default route (`0.0.0.0/0`) handles public Internet breakouts locally, preserving corporate WAN tunnel bandwidth.

---

#### Test DP-03: Overlay MTU & MSS Clamping Verification
* **Objective:** Confirm GRE tunnel encapsulation does not drop packets or cause path failure when transmitting large payloads near standard MTU thresholds.
* **Execution Node:** `BR-EDGE-01`
* **Method:** Cisco IOS Extended Ping Dialog (`size: 1476`, `repeat: 5`)

```text
BR-EDGE-01#ping
Protocol [ip]: 
Target IP address: 10.254.0.1
Repeat count [5]: 5
Datagram size [100]: 1476
Timeout in seconds [2]: 
Extended commands [n]: 
Sweep range of sizes [n]: 
Type escape sequence to abort.
Sending 5, 1476-byte ICMP Echos to 10.254.0.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/2 ms
```
* **Analysis:** Validates 1476-byte unfragmented payload transmission over the GRE overlay, accommodating the 24-byte GRE + IPv4 header budget within a 1500-byte WAN physical MTU.
