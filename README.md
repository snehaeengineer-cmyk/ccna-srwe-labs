# CCNA SRWE Labs

![CCNA Progress](https://img.shields.io/badge/CCNA-ITN%20%E2%9C%93%20SRWE%20%E2%86%92%20ENSA-blue)

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue)
![Routing](https://img.shields.io/badge/Routing-Static%20Routing-success)
![Switching](https://img.shields.io/badge/Switching-VLANs%20%26%20STP-orange)
![Security](https://img.shields.io/badge/Security-Port%20Security-purple)
![Status](https://img.shields.io/badge/Status-Completed-success)


Hands-on Cisco switching and routing labs completed for **Switching, Routing, and Wireless Essentials (SRWE)**, part of the CCNA curriculum (Networks and Protocols / Network Security and Automation coursework).

All configurations were built and verified on physical Cisco gear (2960 switches, ISR4321 routers) in a lab environment, console-cable access via PuTTY.

## What's in here

| Lab | Topics | Devices |
|---|---|---|
| [Lab 1](./lab1-vlan-trunk-intervlan) | VLANs, 802.1Q trunking, router-on-a-stick inter-VLAN routing | 3x Cisco 2960 switches, ISR4321 router |
| [Lab 2](./lab2-stp-hsrp) | Spanning Tree (PVST+ / Rapid PVST+), PortFast, BPDU Guard, HSRP gateway redundancy | 3x Cisco 2960 switches, 2x ISR4321 routers |
| [Lab 3](./lab3-vlan-security-static-routing) | VLAN/trunk refresher, port security, DTP hardening, STP security, DHCP snooping, static routing | 2x Cisco 2960 switches, 2x ISR4321 routers |

Each lab folder contains:
- `README.md` — topology, addressing table, and a walkthrough of the configuration with reasoning
- `*.txt` config files — full running-config-style command sets per device, ready to paste into a CLI or Packet Tracer

## Core skills demonstrated

- VLAN creation, access port assignment, and SVI (management interface) configuration
- 802.1Q trunking with native VLAN control, trunk pruning (`switchport trunk allowed vlan`), and DTP hardening
- Router-on-a-stick inter-VLAN routing using sub-interfaces and `encapsulation dot1Q`
- Spanning Tree Protocol: root bridge election/manipulation, PVST+ vs. Rapid PVST+ convergence behavior, PortFast, BPDU Guard
- First Hop Redundancy Protocol (HSRP): active/standby router roles, priority and preemption, virtual IP/MAC
- Switch port security: MAC address limiting, violation modes, sticky MAC learning
- Layer 2 attack mitigation: DTP disablement, native VLAN hardening, unused port shutdown + black-hole VLAN, DHCP snooping
- Static routing: directly connected, recursive, and default routes; verification via `show ip route`, `traceroute`

## Tools

Cisco IOS CLI (console access via PuTTY), Cisco Packet Tracer / physical lab equipment (2960-24TT switches, ISR4321 routers).

---
*Coursework for the CCNA track (SRWE module), TH Köln Universiy — Prof. Dr. A. Grebe.*
# ccna-srwe-labs
# ccna-srwe-labs
# ccna-srwe-labs
# ccna-srwe-labs
# ccna-srwe-labs
# ccna-srwe-labs
