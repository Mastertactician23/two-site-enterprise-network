# Technical Report
## Two-Site Enterprise Network — Design, Implementation and Failure Testing

**Platform:** Cisco Packet Tracer 8.2.2
**Author:** Asibey-Kitiabi Kofi — IT Support Specialist / System Administrator, CCNA
**Document type:** Design specification, implementation record, and test results

---

## 1. Executive summary

This report documents the design, configuration, and failure testing of a two-site enterprise network comprising a headquarters campus and a branch office connected over a serial WAN link. Each site serves three departments — HR, Sales, and Finance — across a redundant collapsed-core architecture.

The design objective was resilience at every layer where a single component failure would otherwise interrupt service: dual gateways per VLAN, dual uplinks per access switch, dual paths from core to router, and dynamic routing capable of reconverging around a failed link without intervention. The one deliberate exception is the WAN link, discussed in section 9.

The implementation was verified by end-to-end connectivity testing across all VLANs at both sites, followed by a structured failure-injection plan covering gateway failure, core switch failure, spanning-tree reconvergence, routing path failure, WAN outage, and rogue-device attachment.

**Two configuration defects were identified by that testing and by no other means.** Both are documented in section 8. Neither was visible under normal operation; both would have manifested as an outage during a real failure event.

---

## 2. Requirements

| # | Requirement | How it is met |
|---|---|---|
| R1 | Departments logically separated | Three user VLANs per site, enforced at the access layer |
| R2 | No single gateway failure interrupts a department | HSRPv2 across two multilayer switches per site |
| R3 | Access switch survives loss of one core switch | Each access switch dual-homed to both |
| R4 | Core switch survives loss of its router uplink | Dedicated transit VLAN providing a second OSPF path |
| R5 | Routing adapts automatically to link failure | OSPFv2, single area, both sites |
| R6 | Sites interconnected | Serial WAN, 10.0.0.0/30 |
| R7 | Addressing centrally managed | DHCP on each site's router, relayed via SVIs |
| R8 | Management access restricted and encrypted | Dedicated management VLAN, SSHv2 only, ACL on VTY lines |
| R9 | Rogue switches cannot influence topology | PortFast with BPDU Guard on all access ports |

---

## 3. Physical design

### 3.1 Device inventory

| Device | Model | Role |
|---|---|---|
| R-HQ | Cisco 2901 + HWIC-2T | HQ edge router, DHCP server, WAN termination |
| R-BR | Cisco 2901 + HWIC-2T | Branch edge router, DHCP server, WAN termination |
| MLS-HQ1, MLS-HQ2 | Cisco 3560-24PS | HQ collapsed core — inter-VLAN routing, HSRP, OSPF |
| MLS-BR1, MLS-BR2 | Cisco 3560-24PS | Branch collapsed core |
| SW-HQ1/2/3 | Cisco 2960-24TT | HQ access — HR, Sales, Finance |
| SW-BR1/2/3 | Cisco 2960-24TT | Branch access — HR, Sales, Finance |

Forty-two end devices: six workstations and one printer per access switch.

### 3.2 Architectural decisions

**Collapsed core over three-tier.** At this scale a separate distribution layer would add cost and a forwarding hop without adding resilience. The two multilayer switches already provide redundant Layer 3 termination for every VLAN, which is what a distribution layer would otherwise supply.

**Access switches dual-homed.** Each connects to both multilayer switches. Spanning tree blocks one uplink per VLAN, so only one forwards at a time, but the second is available within seconds of a failure. Root bridge placement is split across the pair (section 5.2), so different VLANs block different uplinks and both physical links carry production traffic.

**Routers connected to both core switches over routed links, not trunks.** This keeps the routing boundary explicit and gives OSPF two equal-cost paths from each site's core to its WAN edge. It also means the routers hold no VLAN state, which simplifies the failure domain.

**A dedicated transit VLAN per site.** VLAN 100 carries a /30 between the two core switches, allowing them to form a direct OSPF adjacency rather than reaching each other only through the router. Section 8.2 explains why this proved essential rather than optional.

---

## 4. Logical design

### 4.1 VLAN scheme

The same VLAN IDs are reused at both sites. This is safe because the sites are separated by a routed boundary — VLAN 10 at HQ and VLAN 10 at the branch are distinct broadcast domains in different subnets. Reusing IDs keeps configuration consistent across sites and simplifies extension to a third location.

| VLAN | Name | HQ subnet | Branch subnet | Purpose |
|---|---|---|---|---|
| 10 | HR | 192.168.10.0/24 | 172.16.10.0/24 | HR workstations and printer |
| 20 | SALES | 192.168.20.0/24 | 172.16.20.0/24 | Sales workstations and printer |
| 30 | FINANCE | 192.168.30.0/24 | 172.16.30.0/24 | Finance workstations and printer |
| 99 | MGMT | 192.168.99.0/24 | 172.16.99.0/24 | Switch management interfaces |
| 100 | TRANSIT | 10.1.1.8/30 | 10.2.2.8/30 | Core-to-core OSPF adjacency |
| 999 | PARKING | — | — | Native VLAN, unused-port assignment |

**VLAN 999** carries no traffic by design and serves two purposes. It is the native VLAN on every trunk, so untagged frames arriving on a trunk land in a VLAN with no route out — closing the double-tagging VLAN-hopping path that a default native VLAN of 1 leaves open. It is also the access VLAN for every unused, administratively shut port, so a cable connected to a spare port grants nothing.

**VLAN 99** isolates switch management from user traffic. Combined with the VTY access list (section 7), a workstation in a user VLAN cannot reach a switch's login prompt at all, and therefore cannot attempt authentication against it.

### 4.2 Infrastructure addressing

All infrastructure links use /30 subnets from 10.0.0.0/8, keeping infrastructure addressing visually distinct from user addressing.

| Link | Subnet |
|---|---|
| R-HQ ↔ MLS-HQ1 | 10.1.1.0/30 |
| R-HQ ↔ MLS-HQ2 | 10.1.1.4/30 |
| MLS-HQ1 ↔ MLS-HQ2 (SVI 100) | 10.1.1.8/30 |
| R-BR ↔ MLS-BR1 | 10.2.2.0/30 |
| R-BR ↔ MLS-BR2 | 10.2.2.4/30 |
| MLS-BR1 ↔ MLS-BR2 (SVI 100) | 10.2.2.8/30 |
| R-HQ ↔ R-BR (serial WAN) | 10.0.0.0/30 |

---

## 5. Layer 2 implementation

### 5.1 Trunking

All inter-switch links are 802.1Q trunks with a non-default native VLAN and explicit allowed-VLAN lists.

Explicit allowed lists were used rather than the default of all VLANs. This limits the scope of a misconfiguration, reduces unnecessary broadcast propagation, and makes the intended traffic flow readable from the configuration. The core-to-core trunk additionally carries VLAN 100, which access trunks do not.

A platform constraint applies: the **3560 requires `switchport trunk encapsulation dot1q`** before `switchport mode trunk`, since it supports both ISL and 802.1Q and defaults to auto-negotiation. The 2960 is 802.1Q-only and rejects the command. A mixed-platform design therefore cannot apply a single trunk configuration template across all switches.

### 5.2 Spanning tree

Rapid PVST+ throughout, with root bridge placement set explicitly rather than left to election:

| Site | Root for VLAN 10, 99 | Root for VLAN 20, 30 |
|---|---|---|
| HQ | MLS-HQ1 (secondary MLS-HQ2) | MLS-HQ2 (secondary MLS-HQ1) |
| Branch | MLS-BR1 (secondary MLS-BR2) | MLS-BR2 (secondary MLS-BR1) |

Splitting the root means HR and management traffic takes one uplink from each access switch while Sales and Finance traffic takes the other. Left unconfigured, the election could land on an access switch, producing poor paths; and a single root for all VLANs would leave one uplink from every access switch blocked for everything, idling half the access-layer capacity.

Root placement is deliberately aligned with HSRP active assignment (section 6.1). Where the two disagree, every packet crosses the inter-core link unnecessarily to reach its active gateway.

### 5.3 Access port hardening

Every user-facing port carries PortFast, BPDU Guard, and `switchport nonegotiate`.

PortFast moves the port to forwarding immediately rather than through listening and learning. Beyond convenience, this matters operationally: a workstation booting into a thirty-second STP delay will frequently fail DHCP and fall back to an APIPA address.

BPDU Guard is the control that makes PortFast safe. Any BPDU arriving on a PortFast port err-disables it, preventing an unauthorised switch from joining the spanning tree — and, in the worst case, becoming root and drawing traffic through itself.

`switchport nonegotiate` disables DTP, so a port cannot be negotiated into a trunk by an attacker or a misconfigured device.

---

## 6. Layer 3 implementation

### 6.1 First-hop redundancy

HSRPv2 provides a virtual gateway address per user VLAN. Both multilayer switches hold a physical address in every VLAN; the virtual address is what clients use.

| VLAN | Virtual IP (HQ) | Active | Standby | Priority |
|---|---|---|---|---|
| 10 | 192.168.10.1 | MLS-HQ1 | MLS-HQ2 | 110 / 90 |
| 20 | 192.168.20.1 | MLS-HQ2 | MLS-HQ1 | 110 / 90 |
| 30 | 192.168.30.1 | MLS-HQ2 | MLS-HQ1 | 110 / 90 |
| 99 | 192.168.99.1 | MLS-HQ1 | MLS-HQ2 | 110 / 90 |

Preemption is enabled on all groups, so a recovered switch reclaims its active role automatically rather than requiring manual intervention after every failure event.

Active roles are split across the pair for the same reason as root bridges: to use both devices rather than leaving one idle until an outage.

### 6.2 Routing

OSPF process 1, single area 0, across both sites. A single area is appropriate at this scale — the link-state database is small and reconvergence cost is negligible. Multi-area with the WAN as an area boundary is the natural evolution as sites are added.

Router IDs are assigned manually (1.1.1.1 through 6.6.6.6). Left automatic, OSPF derives the ID from interface addressing and can change it if an interface is removed, which makes neighbour tables harder to read and troubleshooting harder to follow.

`passive-interface default` is applied on the multilayer switches, with the routed uplink and the transit SVI explicitly re-enabled. Without it, the core switches would send OSPF hellos into every user VLAN. This is both wasted traffic and an information disclosure — a packet capture from any user port would reveal the routing topology, adjacencies, and infrastructure addressing.

### 6.3 DHCP

Address pools are held on each site's router, with `ip helper-address` on every user SVI relaying client broadcasts to it. Placing pools on the routers rather than the multilayer switches avoids both HSRP peers responding to the same request with conflicting offers.

Each pool's `default-router` is the **HSRP virtual IP**, not a physical SVI address. This is the detail that makes first-hop redundancy functional for clients; section 8.1 documents what happens when it is wrong.

The first twenty addresses of each subnet are excluded and reserved. Printers are statically addressed within that reserved range so their addresses remain stable for print queues and driver configurations.

---

## 7. Security posture

| Control | Implementation | Threat addressed |
|---|---|---|
| Departmental segmentation | VLANs 10, 20, 30 | Lateral movement between departments |
| Management plane isolation | VLAN 99 with ACL 10 on VTY lines | Device access attempts from user VLANs |
| Encrypted administration | SSHv2; Telnet disabled | Credential capture in transit |
| Local authentication | `login local`, privilege 15 account | Shared-password administration |
| Rogue switch prevention | BPDU Guard on all access ports | Spanning-tree topology manipulation |
| DTP suppression | `switchport nonegotiate` | VLAN hopping via negotiated trunk |
| Native VLAN isolation | Native VLAN 999, otherwise unused | VLAN hopping via double tagging |
| Unused port lockdown | Administratively shut, VLAN 999 | Unauthorised physical connection |

Credentials in the lab file are documented and deliberately simple for reproducibility. A production deployment would use unique per-device credentials managed centrally through TACACS+ or RADIUS rather than local accounts, with local authentication retained only as a fallback.

**Not implemented:** inter-VLAN access control. Departments occupy separate broadcast domains, but traffic between them is routed without filtering. Where business policy requires restriction — for example, preventing general access to Finance resources — ACLs applied inbound on the core SVIs would be the appropriate mechanism.

---

## 8. Findings from testing

Two defects were present in a configuration that passed every connectivity test. Both were found by failure injection and by nothing else.

### 8.1 DHCP advertising a physical gateway address

**Symptom during testing.** Test 1 shut the active HSRP SVI. HSRP transitioned correctly and the standby assumed the active role, but client traffic did not recover.

**Cause.** The DHCP pool's `default-router` was set to a physical SVI address rather than the HSRP virtual IP. Clients had therefore never been informed that the virtual address existed. When the switch holding their configured gateway went down, they had no alternative — despite first-hop redundancy being correctly configured and functioning at the network layer.

**Resolution.** `default-router` set to the virtual IP in every pool.

**Observation.** The redundancy mechanism was working perfectly. The clients simply were not using it. A feature can be correctly configured and still deliver nothing if the component consuming it is pointed elsewhere — and no connectivity test will reveal this, because everything works until the specific failure occurs.

### 8.2 Transit VLAN absent from the core-to-core trunk

**Symptom during testing.** Each core switch reported only one OSPF neighbour — its router — where two were expected. Route tables appeared complete, so the condition was easy to dismiss.

**Cause.** VLAN 100 was not included in the allowed-VLAN list on the trunk between the core switches. The SVI therefore had no active path and remained down, and the adjacency never formed. Route tables looked correct because each switch still had one working path via its router.

**Resolution.** VLAN 100 added to the allowed list at both ends, SVI brought up, and the transit subnet added to the OSPF network statements with the interface made non-passive.

**Observation.** Had this remained, Test 4 would have produced a blackhole rather than a reroute. A core switch losing its router uplink would have had no alternative path, while continuing to advertise itself as the active HSRP gateway and accepting traffic it could not forward. This is worse than an outright failure, because the failure is silent from the client's perspective.

**Secondary observation.** A trunk's allowed-VLAN list must match at both ends. Configuring one side produces no effect and no useful error — the observable symptom is an SVI stuck in `up/down`, which points nowhere obvious.

---

## 9. Known limitations

**Single WAN link.** The serial connection between sites is a single point of failure. This is a deliberate scope decision rather than an oversight, and Test 5 measures its impact precisely: inter-site traffic fails while both sites continue operating internally. A production design would require a second path, ideally from a different carrier.

**No internet edge or perimeter firewall.** Both sites are private networks. Adding an ISP connection with NAT and stateful filtering is the most valuable next extension.

**No inter-VLAN filtering.** Discussed in section 7.

**No EtherChannel.** The topology uses single links. Packet Tracer's port-channel implementation does not reliably propagate allowed-VLAN lists from member ports to the auto-created Port-channel interface, which silently suspends members with a misleading error. Any addition should set trunk configuration explicitly on both the member range and the port-channel interface, and verify agreement in `show running-config`.

**Simulated platform.** Packet Tracer implements a subset of IOS. Convergence timings, hardware forwarding behaviour, and some feature interactions differ from physical equipment — notably, HSRP interface tracking and `no autostate` are unavailable on some images. Results are functional verification, not performance measurement.

---

## 10. Test results

### 10.1 Connectivity verification

| Test | Path | Result |
|---|---|---|
| Intra-VLAN, intra-site | HQ HR PC → HQ HR PC | Pass |
| Inter-VLAN, intra-site | HQ HR PC → HQ Sales PC | Pass — routed at core |
| Gateway reachability | HQ HR PC → 192.168.10.1 | Pass — virtual IP responds |
| Static host reachability | HQ Sales PC → HQ HR printer | Pass |
| WAN transit | HQ HR PC → 10.0.0.2 | Pass |
| Intra-VLAN, inter-site | HQ HR PC → Branch HR PC | Pass |
| Inter-VLAN, inter-site | HQ HR PC → Branch Finance PC | Pass |
| Management across WAN | Branch PC → 192.168.99.11 | Pass |
| Management ACL enforcement | HR PC → SSH to switch | Correctly refused |

### 10.2 Failure injection

| Test | Method | Result |
|---|---|---|
| 1 | Shut active HSRP SVI during continuous ping | Standby assumed active; preemption restored on recovery |
| 2 | Power off one multilayer switch | HSRP, STP and OSPF reconverged; service maintained via peer |
| 3 | Shut forwarding access uplink | Blocked uplink transitioned to forwarding |
| 4 | Shut core-to-router link | Traceroute path shifted to peer core switch via transit VLAN |
| 5 | Shut serial WAN interface | Inter-site failed as expected; intra-site unaffected |
| 6 | Attach switch to access port | Port err-disabled by BPDU Guard |

Traceroute output before and after Test 4 is the most demonstrative single artefact, showing the routing decision changing in direct response to a topology change.

---

## 11. Conclusion

The implementation meets all nine stated requirements. Redundancy is present and verified at the gateway, access uplink, core switch, and routing layers, with observed failover behaviour rather than assumed behaviour.

The two defects recorded in section 8 are the more useful outcome than the successful tests. Both existed in a configuration that passed every functional check, and both would have surfaced first during a genuine outage. They support a general point: a redundancy feature that has never been exercised is an assumption, not a control.

The design's principal weakness — a single WAN link — is documented rather than concealed, and quantified by Test 5. In order of practical priority, the most valuable extensions would be a redundant WAN path, an internet edge with NAT and stateful filtering, and inter-VLAN access control lists where business policy requires them.

---

## Appendix A — Verification command reference

```
show vlan brief                          VLAN membership
show interfaces trunk                    trunk status, native VLAN, allowed list
show spanning-tree vlan <id>             root bridge and port roles
show spanning-tree inconsistentports     native VLAN and PVID conflicts
show interfaces status err-disabled      ports disabled by BPDU Guard
show standby brief                       HSRP roles and virtual IPs
show ip ospf neighbor                    adjacency state
show ip ospf interface brief             which interfaces run OSPF
show ip route ospf                       learned routes
show ip dhcp pool                        pool subnets and utilisation
show ip dhcp binding                     issued leases
show ip interface brief                  interface up/down summary
show controllers serial 0/0/0            DCE or DTE end of a serial link
show cdp neighbors                       verify cabling against the design
```

---

**Asibey-Kitiabi Kofi**
IT Support Specialist · System Administrator · CCNA
[LinkedIn](https://www.linkedin.com/in/asibey-kitiabi/) · [GitHub](https://github.com/Mastertactician23)
