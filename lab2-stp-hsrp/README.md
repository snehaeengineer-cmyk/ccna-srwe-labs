# Lab 2 — Spanning Tree Protocol (STP) & HSRP Gateway Redundancy

## Objective

Build a redundant Layer 2 topology with a switching loop, control which switch becomes STP root, observe and compare PVST+ vs. Rapid PVST+ convergence times, harden access ports with PortFast/BPDU Guard, then add a redundant default gateway using HSRP.

```mermaid
flowchart TD
    %% Define Devices
    Lo1(["☁️ Lo1 (Loopback)"])
    R2(["🌐 R2"])
    R1(["🌐 R1"])
    R3(["🌐 R3"])
    S1["💻 S1"]
    S2["💻 S2"]
    S3["💻 S3"]
    PCA["🖥️ PC-A"]

    %% Loopback connection
    Lo1 --- R2

    %% Router Connections (Core Mesh)
    R2 -- "G0/0/0 <---> G0/0/1" --- R1
    R2 -- "G0/0/1 <---> G0/0/1" --- R3

    %% Router to Switch Uplinks
    R1 -- "G0/0/0 <---> G0/1" --- S1
    R3 -- "G0/0/0 <---> G0/1" --- S3

    %% Switch Mesh Interconnections
    S1 -- "F0/3 <---> F0/3" --- S3
    S1 -- "F0/1" --- S2
    S3 -- "F0/1" --- S2

    %% Host Access Connections
    S1 -- "F0/6" --- PCA

'''

S1–S2–S3 form a triangle (a deliberate Layer 2 loop) so STP has something to block. R1 and R3 both sit on the same LAN as default-gateway candidates for HSRP; R2 simulates an upstream/Internet hop via a loopback interface.

## Addressing Table

| Device | Interface | IP Address | Subnet Mask |
|---|---|---|---|
| R1 | G0/0/0 | 192.168.10.1 | 255.255.255.0 |
| | G0/0/1 | 10.1.1.1 | 255.255.255.252 |
| R2 | G0/0/0 | 10.1.1.2 | 255.255.255.252 |
| | G0/0/1 | 10.2.2.2 | 255.255.255.252 |
| | Lo1 | 209.165.200.225 | 255.255.255.224 |
| R3 | G0/0/0 | 192.168.10.3 | 255.255.255.0 |
| | G0/0/1 | 10.2.2.1 | 255.255.255.252 |
| S1/S2/S3 | VLAN 99 | .11 / .12 / .13 | 255.255.255.0 |
| PC-A | NIC | 192.168.10.11 | 255.255.255.0 |
| PC-C | NIC | 192.168.10.33 | 255.255.255.0 |

VLANs: **10 (User)**, **99 (Management, native)**. HSRP virtual IP: **192.168.10.254**.

## Approach

**1. Build the loop on purpose.** S1, S2, and S3 are deliberately fully meshed (every switch trunks to every other switch) so that Spanning Tree has real work to do — without STP, this topology would broadcast-storm itself into uselessness.

**2. Root bridge: default vs. forced.** First let STP elect a root bridge by default behavior (lowest bridge ID — priority + MAC address, so with equal default priority it comes down to the lowest MAC). Recorded that result with `show spanning-tree`, then overrode it deliberately: `spanning-tree vlan 1 root primary` on S2 made it root for all VLANs, and `root secondary` on S1 set it up as the designated backup. This is the practical reason you'd ever touch STP priority by hand — leaving root election to chance means whichever switch happens to have the lowest MAC address controls your topology's logical shape, which is not something you want to discover during an outage.

**3. Watching convergence happen, not just trusting the spec.** Using `debug spanning-tree events`, forced a topology change (shut/no-shut a link between S2 and S3) and timestamped how long it took the network to re-converge — once under legacy PVST+, once after switching every switch to Rapid PVST+ (`spanning-tree mode rapid-pvst`). PVST+ uses the original 802.1D timers (max-age/forward-delay), which is where the familiar ~30–50 second convergence figure comes from; Rapid PVST+ (802.1w) replaces blocking/listening/learning with an explicit proposal/agreement handshake between switches, which is why it converges in roughly a second instead of tens of seconds.

**4. PortFast — but only where it's safe.** PortFast skips the listening/learning STP delay on a port, but it's only safe on ports that can never have another switch plugged into them (i.e. real end-host access ports), since skipping STP validation on a port facing another switch is exactly how an accidental or malicious loop gets introduced. Applied PortFast only to the access ports actually carrying PCs (S1 and S3), never to inter-switch trunk ports.

**5. HSRP for default gateway redundancy.** PC-A and PC-C each only know about one default gateway IP. If R1 — physically connected as their gateway — went down, both PCs would lose all off-subnet connectivity with zero ability to reroute, despite R3 sitting right there as a perfectly viable backup path. HSRP solves this by presenting a single virtual IP/MAC that PCs use as their gateway, backed by two real routers: R1 configured as active (`standby 1 priority 150` + `standby 1 preempt` so it reliably reclaims the active role once it comes back up), R3 as standby. Verified the failover by pinging continuously from PC-A through R2's loopback while manually shutting the link to the active router — packet loss was brief and bounded to the HSRP hold timer, rather than indefinite.

## Verification & key findings

- Default STP root election was determined purely by bridge ID (lowest MAC address wins ties at equal priority) — not by link speed, position in the topology, or anything administrator-controlled, which is precisely why production networks set root bridge priority explicitly instead of leaving it to chance.
- Rapid PVST+ converged dramatically faster than legacy PVST+ for the identical topology change, confirmed by debug timestamp deltas rather than just citing the spec numbers.
- `show standby` confirmed a single virtual MAC address shared by both HSRP routers, with the active router showing the configured (higher) priority and the standby showing the lower one — `show standby brief` gives the same status in a single summary line but omits the timers and state-change history that the full `show standby` output includes.
- After moving PC-A/PC-C's gateway to the HSRP virtual IP, killing the link to the active router caused only a short, bounded ping interruption before traffic resumed via the standby router — confirming gateway failover actually works, not just that it's configured.

## Reflection

- **Rapid PVST+'s main benefit:** convergence in roughly a second via explicit proposal/agreement handshakes between switches, instead of the 30–50 second blocking→listening→learning→forwarding wait of legacy 802.1D STP.
- **Why PortFast speeds convergence:** it puts an access port straight into the forwarding state, skipping the listening and learning delays entirely — appropriate only because a true end-host port can't create a loop.
- **BPDU Guard as a DoS defense:** if a port configured with PortFast ever receives a BPDU (the control frame switches use to negotiate STP), that's a signal something unexpected — another switch, or an attacker — has been connected to what should be an end-host port. BPDU Guard immediately disables (err-disables) that port rather than letting it participate in STP, which prevents an attacker from injecting BPDUs to manipulate root election or trigger forced topology recalculations as a denial-of-service vector.

## Files

- [`s1-config.txt`](./s1-config.txt) — Switch S1 (STP secondary root, PortFast)
- [`s2-config.txt`](./s2-config.txt) — Switch S2 (STP primary root)
- [`s3-config.txt`](./s3-config.txt) — Switch S3 (PortFast)
- [`r1-config.txt`](./r1-config.txt) — Router R1 (HSRP active)
- [`r3-config.txt`](./r3-config.txt) — Router R3 (HSRP standby)
