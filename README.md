# Two-Site Enterprise Network — Cisco Packet Tracer

A headquarters campus and a branch office, connected over a serial WAN, built and failure-tested in Cisco Packet Tracer 8.2.2.

![Platform](https://img.shields.io/badge/Cisco-Packet%20Tracer%208.2.2-1BA0D7)
![Focus](https://img.shields.io/badge/Focus-Routing%20%7C%20Switching%20%7C%20Redundancy-informational)
![Status](https://img.shields.io/badge/Status-Built%20%26%20Tested-success)

---

## Overview

Twelve network devices and forty-two endpoints across two sites, each serving HR, Sales, and Finance departments. Every layer where a single failure would interrupt service has a second path: dual gateways, dual access uplinks, dual core-to-router links, and dynamic routing that reconverges without intervention.

The configuration is the smaller half of the work. The larger half is the failure-injection test plan — six tests, each with recorded before-and-after output, verifying that the redundancy actually functions rather than merely being present in the config.

Two of those tests found real defects in my own configuration. Both are documented below.

---

## Topology

```
  BUILDING 1 — HEADQUARTERS            BUILDING 2 — BRANCH OFFICE

  SW-HQ1   SW-HQ2   SW-HQ3             SW-BR1   SW-BR2   SW-BR3
   (HR)   (Sales) (Finance)             (HR)   (Sales) (Finance)
     \   /   \   /   \   /                \   /   \   /   \   /
      \ /     \ /     \ /                  \ /     \ /     \ /
      / \     / \     / \                  / \     / \     / \
     /   \   /   \   /   \                /   \   /   \   /   \
  +---------+   +---------+           +---------+   +---------+
  | MLS-HQ1 |===| MLS-HQ2 |           | MLS-BR1 |===| MLS-BR2 |
  |  3560   |   |  3560   |           |  3560   |   |  3560   |
  +----+----+   +----+----+           +----+----+   +----+----+
       |             |                     |             |
       +------+------+                     +------+------+
              |                                   |
         +----+----+     Serial WAN          +----+----+
         |  R-HQ   |o~~~~~~~~~~~~~~~~~~~~~~~o|  R-BR   |
         |  2901   |      10.0.0.0/30        |  2901   |
         +---------+                         +---------+
```

Each access switch is dual-homed to both core switches. Spanning tree blocks one uplink per VLAN — but because root bridge placement is split across the pair, different VLANs block different uplinks, so both physical links carry production traffic rather than one sitting idle.

The `===` link between core switches carries VLAN 100, a dedicated transit subnet that lets the pair peer directly over OSPF. Without it, a core switch that loses its uplink to the router has no alternative path and blackholes traffic while still advertising itself as the active gateway.

---

## Addressing

**Headquarters**

| VLAN | Name | Subnet | Gateway (HSRP VIP) | Active |
|---|---|---|---|---|
| 10 | HR | 192.168.10.0/24 | 192.168.10.1 | MLS-HQ1 |
| 20 | SALES | 192.168.20.0/24 | 192.168.20.1 | MLS-HQ2 |
| 30 | FINANCE | 192.168.30.0/24 | 192.168.30.1 | MLS-HQ2 |
| 99 | MGMT | 192.168.99.0/24 | 192.168.99.1 | MLS-HQ1 |
| 100 | TRANSIT | 10.1.1.8/30 | — | — |

**Branch office**

| VLAN | Name | Subnet | Gateway (HSRP VIP) | Active |
|---|---|---|---|---|
| 10 | HR | 172.16.10.0/24 | 172.16.10.1 | MLS-BR1 |
| 20 | SALES | 172.16.20.0/24 | 172.16.20.1 | MLS-BR2 |
| 30 | FINANCE | 172.16.30.0/24 | 172.16.30.1 | MLS-BR2 |
| 99 | MGMT | 172.16.99.0/24 | 172.16.99.1 | MLS-BR1 |
| 100 | TRANSIT | 10.2.2.8/30 | — | — |

VLAN IDs are reused across sites — safe, because the two sites are separated by a routed boundary. Infrastructure links use `10.x.x.x/30` so they remain visually distinct from user addressing at a glance.

VLAN 999 exists at both sites and carries nothing by design. It is the native VLAN on every trunk, and the assigned VLAN for every unused, shut-down port.

---

## Implemented

**Layer 2**
- Six VLANs per site, including a dedicated management VLAN and an unused-port parking VLAN
- 802.1Q trunking with non-default native VLAN and explicit allowed-VLAN lists
- Rapid PVST+ with root bridge placement split across the core pair and aligned to HSRP active assignment
- PortFast with BPDU Guard on every access port; DTP disabled via `switchport nonegotiate`

**Layer 3**
- Inter-VLAN routing on SVIs at the collapsed core
- HSRPv2 with preemption, active roles load-shared across both core switches
- OSPFv2 single area 0 spanning both sites, manual router IDs, `passive-interface default` on the core switches
- Dedicated transit VLAN per site for direct core-to-core OSPF adjacency
- Point-to-point routed links from each core switch to its site router
- Serial WAN with clock rate on the DCE end

**Services**
- DHCP pools on each site's router, relayed from user SVIs via `ip helper-address`
- Pool default-gateway set to the HSRP virtual IP
- Static addressing for printers, within the DHCP-excluded range

**Management**
- Dedicated management VLAN with SVIs on every switch
- SSHv2 with local authentication; Telnet disabled
- Standard ACL on VTY lines restricting administrative access to the management subnets

---

## Failure testing

| # | Test | Method | Verifies |
|---|---|---|---|
| 1 | Gateway failover | Shut the active HSRP SVI during a continuous ping | Standby assumes the role; preemption restores on recovery |
| 2 | Core switch failure | Power off a multilayer switch entirely | HSRP, STP and OSPF reconverge together |
| 3 | STP reconvergence | Shut the forwarding access uplink | Blocked port transitions to forwarding |
| 4 | Routing path failure | Shut a core-to-router link | Traceroute path shifts via the peer core switch |
| 5 | WAN failure | Shut the serial interface | Inter-site traffic fails; intra-site unaffected |
| 6 | Rogue switch | Attach a switch to an access port | BPDU Guard err-disables the port |

Procedures and expected output are in [`CONFIG-GUIDE.txt`](CONFIG-GUIDE.txt), Phase 12.

---

## What testing found

Two defects surfaced only because the tests were run. Both were invisible in normal operation.

**The DHCP pools were handing out a physical switch address as the default gateway.** Every connectivity test passed. HSRP was configured correctly and failed over correctly. But clients had never been told the virtual address existed, so when the active switch was shut down in Test 1, they lost their gateway anyway. One line of configuration, and it rendered the entire first-hop redundancy design decorative.

**The core-to-core transit VLAN was not carried on the trunk between the core switches.** Route tables looked complete, because each core switch had a working path through its router. What was missing was the second path. Test 4 would have blackholed traffic rather than rerouting — the switch would have kept advertising itself as the active gateway for a VLAN it could no longer forward out of.

Neither would have been caught by a connectivity test. Both are the kind of defect that surfaces during a real outage rather than during a build.

---

## Repository contents

```
├── README.md
├── CONFIG-GUIDE.txt            Step-by-step build, full configs, checkpoints
├── TECHNICAL-REPORT.md         Design rationale, security posture, results
├── BEGINNER-GUIDE.md           Plain-language explanation of every concept
├── enterprise-network.pkt      Packet Tracer file
├── configs/                    Per-device running configurations
└── docs/
    ├── topology.png
    └── test-results.md
```

---

## Running it

1. Open `enterprise-network.pkt` in Packet Tracer 8.2.2 or later.
2. Allow roughly 60 seconds for STP and OSPF to converge.
3. From any HQ workstation, confirm a DHCP lease, then ping a branch workstation.
4. Work through the failover tests in Phase 12 of the config guide.

Lab credentials are documented in the config guide. They are deliberately simple for reproducibility and should not be treated as a password standard.

---

## Platform notes

Details that cost time and are not obvious from documentation:

**The 3560 and 2960 disagree about trunk encapsulation.** The 3560 requires `switchport trunk encapsulation dot1q` before `switchport mode trunk`, because it supports both ISL and 802.1Q and defaults to auto. The 2960 is 802.1Q-only and rejects the command outright. Mixed-platform designs cannot use a single trunk template.

**`logging synchronous` is effectively mandatory.** Without it, console log messages interrupt input mid-command and silently corrupt what you typed. Correct commands fail with no visible cause. It belongs on every device's console line before anything else is configured.

**A trunk's allowed-VLAN list must match at both ends.** Changing one side alone does nothing, and the resulting symptom — an SVI stuck `up/down` — points nowhere useful.

---

## Known limitations

- **The WAN link is a single point of failure.** A deliberate scope decision; Test 5 measures precisely what is lost when it fails. Production would require a second carrier path.
- **No internet edge, NAT, or perimeter firewall.** Both sites are private networks. This is the most obvious next extension.
- **No EtherChannel.** The topology uses single links. Packet Tracer's port-channel implementation does not reliably propagate allowed-VLAN lists from member ports to the auto-created Port-channel interface, which silently suspends members.
- **No inter-VLAN access control.** Departments are separated into different broadcast domains but traffic between them is not filtered. ACLs on the core SVIs would be the natural addition.
- **Packet Tracer models a subset of IOS.** Convergence timings and hardware forwarding behaviour differ from physical equipment. Results are functional verification, not performance measurement.

---

## Why this project

Network fundamentals underpin most of what security monitoring actually observes. An alert about unexpected east-west traffic is difficult to triage without being able to say why that traffic should or should not cross a particular boundary, and what the normal path between two subnets looks like.

This project is the infrastructure counterpart to the detection and tooling work in my other repositories.

---

## Skills demonstrated

Network design · IP addressing and subnetting · VLANs and 802.1Q trunking · STP root placement and edge hardening · HSRP first-hop redundancy · Layer 3 switching and inter-VLAN routing · OSPFv2 · Serial WAN configuration · DHCP and DHCP relay · SSH hardening and management ACLs · Structured failure testing · Technical documentation

---

**Asibey-Kitiabi Kofi**
IT Support Specialist · System Administrator · CCNA
[LinkedIn](https://www.linkedin.com/in/asibey-kitiabi/) · [GitHub](https://github.com/Mastertactician23)
