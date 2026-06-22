# Lab 1 — VLANs, 802.1Q Trunks & Inter-VLAN Routing

## Objective

Build a switched LAN with multiple VLANs, connect switches together with 802.1Q trunks, and route between VLANs using router-on-a-stick on a Cisco ISR4321.

## Topology

```mermaid
flowchart TD
    %% Define Devices
    R1(["🌐 R1 (ISR4321)"])
    S1["💻 S1 (2960)"]
    S2["💻 S2 (2960)"]
    S3["💻 S3 (2960)"]

    %% Core Router Link
    R1 -- "G0/0/0 <---> G0/1" --- S1

    %% Inter-Switch Trunk Links (Dashed)
    S1 -. "F0/3 <---> F0/3" .-> S3
    S1 -. "F0/1 <---> F0/1" .-> S2
    S2 -. "F0/2 <---> F0/2 (STP Blocked)" .-> S3

    %% VLAN 10 Group A
    subgraph VLAN10_A [VLAN 10]
        PCA["🖥️ PC-A"]
    end
    S1 -- "F0/6" --- PCA

    %% VLAN 10 Group B
    subgraph VLAN10_B [VLAN 10]
        PCB["🖥️ PC-B"]
    end
    S3 -- "F0/11" --- PCB

    %% VLAN 20 Group
    subgraph VLAN20 [VLAN 20]
        PCC["🖥️ PC-C"]
    end
    S3 -- "F0/18" --- PCC

    ```
    

S1 trunks to both S2 and S3 (mesh of switches), with S1's G0/1 uplinked to R1 for inter-VLAN routing. PC-A sits in VLAN 10 off S1, PC-C sits in VLAN 20 off S3.

## Addressing Table

| Device | Interface | IP Address | Subnet Mask | Default Gateway |
|---|---|---|---|---|
| R1 | G0/0/0 | 192.168.99.1 | 255.255.255.0 | N/A |
| | G0/0/0.10 | 192.168.10.1 | 255.255.255.0 | N/A |
| | G0/0/0.20 | 192.168.20.1 | 255.255.255.0 | N/A |
| S1 | VLAN 99 | 192.168.99.11 | 255.255.255.0 | 192.168.99.1 |
| S2 | VLAN 99 | 192.168.99.12 | 255.255.255.0 | 192.168.99.1 |
| S3 | VLAN 99 | 192.168.99.13 | 255.255.255.0 | 192.168.99.1 |
| PC-A | NIC | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| PC-C | NIC | 192.168.20.20 | 255.255.255.0 | 192.168.20.1 |

VLANs: **10 (Student)**, **20 (Faculty)**, **99 (Management, native VLAN on all trunks)**.

## Approach

**1. Switch baseline.** Every switch got the same hardening baseline before any VLAN work: disabled DNS lookup (`no ip domain-lookup`) so mistyped commands don't hang waiting on a lookup, console password + `login`, `enable secret`, `service password-encryption`, and `logging synchronous` so console output doesn't interleave with typed commands. Config saved to NVRAM at the end of each step.

**2. VLANs first, then ports.** Created VLANs 10/20/99 on all three switches before touching interfaces — a VLAN has to exist in the database before a port can be assigned to it. Access ports (`switchport mode access` + `switchport access vlan X`) went on PC-facing interfaces; everything between switches became a trunk.

**3. Trunking with a controlled native VLAN.** All inter-switch links (`S1↔S2`, `S2↔S3`, `S1↔S3`) were set to `switchport mode trunk` explicitly rather than relying on DTP, with VLAN 99 as the native VLAN and only VLANs 10/20/99 permitted (`switchport trunk allowed vlan`). Pinning the native VLAN matters: an untagged frame on a trunk is implicitly placed into whatever VLAN is configured as native on that port, so if the two ends disagree, traffic silently leaks between VLANs. Restricting the allowed list also keeps broadcast traffic for unused VLANs off links that don't need it.

**4. SVI for management, not VLAN 1.** Rather than leaving management on the default VLAN 1 (the out-of-the-box VLAN every port inherits, and consequently an easy target), the management SVI was deliberately moved to VLAN 99 with a dedicated subnet and default gateway pointed at R1.

**5. Router-on-a-stick.** Since a single physical router interface needs to carry traffic for multiple VLANs, R1's G0/0/0 was split into sub-interfaces — one per non-native VLAN (`G0/0/0.10`, `G0/0/0.20`), each with `encapsulation dot1Q <vlan-id>` and its own gateway IP, while the native VLAN (99) gets its IP directly on the physical interface (native/untagged traffic never carries a dot1Q tag, so it can't be matched on a sub-interface). The physical interface is brought up last, after all sub-interfaces are configured — until then, no traffic for any VLAN moves.

## Verification & key findings

- `show ip route` on R1 confirms three directly-connected subnets (192.168.10.0/24, .20.0/24, .99.0/24), each shown twice — once as `C` (connected network) and once as `L` (local, the router's own interface address as a /32 host route). **3** networks directly connected, **0** static routes — everything here is reachable purely through directly-connected sub-interfaces.
- PC-A (VLAN 10) **cannot** ping PC-C (VLAN 20) until the router's sub-interfaces are live — VLANs are separate broadcast domains by design, and Layer 2 switching alone has no path between them. Only a Layer 3 device (the router) can bridge that gap.
- Once router-on-a-stick is fully configured: PC-A ↔ S1 succeeds, and PC-A ↔ PC-C succeeds — confirming inter-VLAN routing is working end-to-end.

## Reflection

- **Security benefit of VLANs:** network segmentation limits the blast radius of a compromised host — devices in different VLANs can't talk directly without passing through a router (and whatever ACL/firewall policy lives there), so a compromised PC in VLAN 10 doesn't have direct Layer 2 reach into VLAN 20.
- **Broadcast domain benefit:** broadcast traffic (ARP requests, DHCP discovers, etc.) stays contained within a VLAN instead of flooding the entire physical switch fabric, which keeps broadcast domains small and traffic overhead down as the network grows.
- **Router-on-a-stick advantage:** uses a single physical interface (and single cable) to serve many VLANs, which is both cost-efficient and easy to scale — adding a new VLAN is just a new sub-interface, not new hardware.

## Files

- [`s1-config.txt`](./s1-config.txt) — Switch S1 full configuration
- [`s2-config.txt`](./s2-config.txt) — Switch S2 full configuration
- [`s3-config.txt`](./s3-config.txt) — Switch S3 full configuration
- [`r1-config.txt`](./r1-config.txt) — Router R1 full configuration (router-on-a-stick)
