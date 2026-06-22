# Lab 3 — VLAN/Trunk Refresher, Switch LAN Security & Static Routing

## Objective

Three-part lab: rebuild a VLAN/trunk/inter-VLAN-routing setup as a refresher, then harden it with switch port security, secure trunking, STP attack mitigation and DHCP snooping, and finally extend the topology with serial WAN links and static routing across three routers.

## Topology

```
                         S0/1/0   R2   S0/1/1
                       +-----------------------+
                       |        ISR4321        |
                       +-----------------------+
                      /                         \
              S0/1/0 /                           \ S0/1/0
            +---------+                         +---------+
            |   R1    |                         |   R3    |
            +---------+                         +---------+
           G0/0/0|                                  |G0/0/0
            +-----+--+                          +----+-----+
            |   S1    |---- VLAN 99 trunk ------|    S2    |
            +---------+                          +---------+
           /         \                          /         \
       [PC-1]      [PC-2]                    [PC-3]      [PC-4]
       VLAN 10                                            VLAN 20
```

R2 sits between R1 and R3 over serial WAN links and simulates an ISP/core hop via a loopback interface, mirroring how a branch site routes through a central hub to reach "the Internet."

## Addressing Table (final, all 3 parts combined)

| Device | Interface | IP Address | Subnet Mask |
|---|---|---|---|
| R1 | G0/0/0.10 | 192.168.10.1 | 255.255.255.0 |
| | G0/0/0.20 | 192.168.20.1 | 255.255.255.0 |
| | G0/0/0.99 | 192.168.99.1 | 255.255.255.0 |
| | S0/1/0 (DCE) | 10.1.1.1 | 255.255.255.252 |
| R2 | S0/1/0 | 10.1.1.2 | 255.255.255.252 |
| | S0/1/1 (DCE) | 10.2.2.2 | 255.255.255.252 |
| | Lo1 | 209.165.200.225 | 255.255.255.224 |
| R3 | G0/0/0.10 | 192.168.10.3 | 255.255.255.0 |
| | G0/0/0.20 | 192.168.20.3 | 255.255.255.0 |
| | S0/1/0 | 10.2.2.1 | 255.255.255.252 |
| S1 / S2 | VLAN 99 | .101 / .102 | 255.255.255.0 |
| PC-1 | NIC | 192.168.10.11 | 255.255.255.0 |
| PC-3 | NIC | 192.168.20.22 | 255.255.255.0 |

VLANs: **10 (Admin)**, **20 (Sales)**, **99 (Management)**, plus **100 (NativeVLAN, unused/empty)** and **999 (BlackHole, unused ports)** added in the security part.

## Approach

### Part 1 — VLAN/Trunk Refresher
Same pattern as Lab 1: VLANs created before ports assigned to them, trunk links between S1/S2 carrying VLAN 99 natively, router-on-a-stick at R1 with one sub-interface per VLAN. This part is a deliberate repeat of earlier material — the curriculum re-tests it because Layer 2 fundamentals are the foundation everything else in this lab builds on.

### Part 2 — Switch LAN Security
This is the core of the lab and covers four independent hardening layers:

**1. Unused VLANs as a trap, not a feature.** VLAN 100 ("NativeVLAN") is created but deliberately left with *no* access ports — its only job is to be the native VLAN on every trunk. VLAN 999 ("BlackHole") is likewise empty and exists purely as a dumping ground for ports that shouldn't be passing traffic at all. Separating "the native VLAN" from any VLAN actually carrying user traffic closes a classic VLAN-hopping vector: if an attacker crafts a double-tagged 802.1Q frame, the outer tag gets stripped by the first switch (because it matches the trunk's native VLAN) and the inner tag can let the frame jump into a VLAN it was never authorized to reach — but only if the native VLAN is *also* a VLAN with real hosts on it. Make the native VLAN a wasteland and there's nothing useful for that hop to land in.

**2. Trunk hardening, explicitly.** `switchport nonegotiate` was applied to every trunk port to disable DTP outright. DTP is dangerous on a production network because it lets a port that's merely *plugged into* a switch automatically negotiate itself into a trunk — meaning an attacker with physical or logical access to an access port could potentially get the switch to autonegotiate a trunk and then see traffic for every VLAN on that link, not just the one VLAN an access port would normally expose. Trunks were also explicitly pruned to only the VLANs that need to traverse them (rather than trusting the "all VLANs allowed by default" behavior) and pinned to native VLAN 100 for the reason above.

**3. Unused ports get shut down and isolated — both.** Every unused switchport on S1 was administratively shut down (`shutdown`) *and* reassigned to the black-hole VLAN 999. Shutting the port down stops it from being used at all; moving it to an isolated VLAN means that even if someone manually re-enabled it later (or a misconfiguration brought it back up), it still wouldn't have a path to anything meaningful. Belt and suspenders, not either/or.

**4. Port security with sticky learning and lenient-but-logged violation handling.** Active access ports got `switchport port-security` with a max of 4 learned MAC addresses, `switchport port-security mac-address sticky` so legitimately-learned MACs get written into the running config automatically (useful for audit trails — you can see exactly which MAC was learned where, without static pre-provisioning every address by hand), and one port (F0/2) got a statically pinned MAC as an example of fully locking a port to one known device. Violation mode was set to `restrict`: drop frames from any MAC beyond the limit and log a Syslog entry, but leave the port itself up — `shutdown` mode would be more aggressive (administratively disables the port on any violation, which means a single rogue device knocks out the whole port until manually re-enabled) and `protect` drops silently with no log at all, which would defeat the point of wanting visibility into violations.

**5. STP and DHCP-specific hardening.** PortFast + BPDU Guard went on the access ports for the same reasons as Lab 2. For DHCP snooping, trunk ports were marked `trusted` (since DHCP server replies legitimately arrive there), untrusted (access) ports were rate-limited to 5 DHCP packets/sec to blunt a DHCP-exhaustion flood, and DHCP snooping was enabled globally plus per-VLAN on S2. The mechanism: DHCP snooping builds a binding table of which IP was leased to which MAC on which port, and blocks DHCP server-type messages (offers, acks) arriving from any port not marked trusted — preventing a rogue or accidental DHCP server on an access port from handing out bogus addressing or executing a man-in-the-middle by becoming a client's "default gateway."

### Part 3 — Static Routing
With switching settled, the topology extended outward with point-to-point serial links (R1↔R2↔R3) and a loopback on R2 standing in for an upstream network. Static routes had to be configured by hand on every router because there's no dynamic routing protocol running:
- **R1 and R3** each got a single static default route pointing out their serial interface — simplest possible design for a "branch" router that only needs to reach "everything else" through one path.
- **R2**, sitting in the middle, needed more precision: a static default route out its loopback (simulating "the rest of the Internet"), plus *recursive* static routes back toward each branch's VLAN subnets (10/99 via R1, 20 via R3) — recursive because the route specifies a next-hop IP rather than an exit interface, requiring the router to do a second lookup to resolve that next-hop to an actual outgoing interface.

## Verification & key findings

- `show vlan brief` after the security hardening pass confirmed only the intended ports remained in VLAN 1 — everything else had been explicitly reassigned to a real VLAN or to the black-hole VLAN, eliminating VLAN 1 as an implicit catch-all.
- `show port-security interface f0/3` after pinging from PC-2 confirmed the switch had learned and recorded PC-2's MAC address via sticky learning, visible directly in the port security state — concrete evidence the configuration was capturing real host identity, not just accepting any traffic.
- At R2, `show ip route` showed only the directly-connected and locally-originated networks until the recursive static routes were added — the VLAN subnets behind R1 and R3 were genuinely unreachable from R2's perspective until those routes existed, which is the expected behavior for a router with no dynamic routing protocol and no default knowledge of anything beyond its own interfaces.
- End-to-end connectivity (PC-1 → R1 serial interface, PC-1 → R2 loopback, PC-3 → R2 loopback) was confirmed only after all three routers had their static routes in place — partial routing tables produce partial reachability, which is a useful debugging signal when something doesn't ping.
- Forward and return traceroute paths between PC-1/PC-3 and R2's loopback were compared explicitly; asymmetric routing here is expected given R2's distinct recursive static routes for VLAN 10 (via R1) versus VLAN 20 (via R3) rather than a symmetric design.

## Reflection (Switch Security)

- **Why shut off all unused ports:** an unused but enabled port is a free, untracked entry point into the network for anyone with physical access — disabling it removes that path entirely rather than relying on someone noticing unauthorized use after the fact.
- **Bulk interface configuration:** `interface range` lets you apply identical configuration to many ports in one command block instead of repeating it interface-by-interface, which matters once you're hardening dozens of unused ports at once.
- **Where MAC limiting matters most:** access ports facing end-user devices, where you have a reasonable expectation of "one PC, one or two MACs" — applying it to a trunk wouldn't make sense, since trunks legitimately carry traffic from many MAC addresses across multiple VLANs.
- **Violation response options:** `protect` (drop silently, no log), `restrict` (drop and log a Syslog/SNMP event, port stays up), `shutdown` (err-disable the entire port on any violation, the strictest option).
- **Why DTP is dangerous on trunk ports:** it lets a port auto-negotiate into trunk mode based on what's plugged into it, which means a switch can be tricked into trunking with a device that has no business carrying every VLAN's traffic.
- **Other trunk security measures:** explicitly pruning the allowed VLAN list (rather than allowing all VLANs by default) and assigning an unused VLAN as the native VLAN, so untagged frames don't land somewhere meaningful.
- **STP man-in-the-middle mechanism:** an attacker connects a rogue switch to an access port and sends BPDUs claiming a superior (lower) bridge priority, tricking the real switches into electing the rogue device as root — once root, it sits in the path of inter-switch traffic and can intercept or manipulate it. BPDU Guard on access ports closes this off by disabling the port the instant a BPDU is ever seen there.
- **DHCP snooping's protection:** stops an unauthorized or accidental DHCP server connected to an untrusted port from successfully handing out IP configuration, which blocks both simple misconfiguration chaos and deliberate man-in-the-middle attacks where a rogue DHCP server hands out itself as the default gateway.

## Files

- [`s1-config.txt`](./s1-config.txt) — Switch S1 (VLANs, trunk security, port security, PortFast/BPDU Guard, DHCP snooping trusted/rate-limit)
- [`s2-config.txt`](./s2-config.txt) — Switch S2 (VLANs, trunk security, DHCP snooping enabled)
- [`r1-config.txt`](./r1-config.txt) — Router R1 (router-on-a-stick + serial default route)
- [`r2-config.txt`](./r2-config.txt) — Router R2 (serial links, loopback, recursive static routes)
- [`r3-config.txt`](./r3-config.txt) — Router R3 (router-on-a-stick + serial default route)
