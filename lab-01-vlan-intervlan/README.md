# Lab 1 — VLAN Segmentation with Inter-VLAN Routing

## Overview
This lab builds a segmented enterprise LAN using VLANs, 802.1Q trunking, and router-on-a-stick inter-VLAN routing, with per-VLAN DHCP and a security-focused ACL isolating the Guest network from internal VLANs. Built and verified in Cisco Packet Tracer.

## Objectives
- Segment a flat network into 5 functional VLANs
- Configure 802.1Q trunking between a core switch and two access switches
- Implement inter-VLAN routing using router-on-a-stick (subinterfaces)
- Provide dynamic IP addressing per VLAN via router-based DHCP
- Isolate the Guest VLAN from internal VLANs using an extended ACL
- Verify the build end-to-end using CLI show commands and live ping tests

## Topology
                Router (R1)
                5 subinterfaces
                     |
                Core Switch
          trunk: VLANs 10,20,30,40,99
                /            \
      Access Switch 1    Access Switch 2
       (VLAN 10, 30)      (VLAN 20, 40)
          |    |              |    |
      PCs VLAN10 VLAN30   PCs VLAN20 VLAN40

      
One router performs router-on-a-stick routing over a single trunk link to the core switch, which trunks down to two access switches. End devices are distributed across VLANs 10, 20, 30, and 40. VLAN 99 is reserved as the native/management VLAN — no end devices are placed on it.

## VLAN and Addressing Plan

| VLAN ID | Name | Subnet | Gateway (router subinterface) |
|---------|------|--------|-------------------------------|
| 10 | Data | 10.10.0.0/24 | 10.10.0.1 |
| 20 | Voice | 10.20.0.0/24 | 10.20.0.1 |
| 30 | IoT | 10.30.0.0/24 | 10.30.0.1 |
| 40 | Guest | 10.40.0.0/24 | 10.40.0.1 |
| 99 | Management (native, no hosts) | 10.99.0.0/24 | 10.99.0.1 |

## Key Configuration

- **Trunking:** All inter-switch and switch-to-router links configured as 802.1Q trunks, explicitly carrying VLANs 10, 20, 30, 40, 99, with VLAN 99 set as the native VLAN (never left as default VLAN 1, to prevent native VLAN mismatch / VLAN-hopping exposure).
- **Router-on-a-stick:** A single physical router interface (g0/0) split into 5 logical subinterfaces, each tagging its own VLAN via `encapsulation dot1Q`, each acting as that VLAN's default gateway.
- **DHCP:** Router-based DHCP pools for VLANs 10, 20, 30, and 40, each excluding the gateway address range. No DHCP pool exists for VLAN 99 since it carries no end devices.
- **Guest isolation ACL:** An extended ACL applied inbound on the Guest subinterface (`g0/0.40 in`) permits DHCP traffic (UDP 67/68) unconditionally, denies traffic from the Guest subnet toward the other four VLAN subnets, and permits everything else (e.g. future internet-bound traffic).

Full CLI configuration is in [`configs/`](./configs).

## Verification

| Check | Result |
|-------|--------|
| `show vlan brief` | All 5 VLANs present; access ports mapped to correct VLANs |
| `show interfaces trunk` | All trunks active, native VLAN = 99, all 5 VLANs allowed |
| `show ip interface brief` (router) | All 5 subinterfaces up/up |
| `show ip dhcp binding` | All PCs in VLANs 10/20/30/40 received correct leases |
| Same-VLAN ping | Success |
| Cross-VLAN ping (e.g. VLAN 10 ↔ VLAN 20) | Success — confirms inter-VLAN routing |
| Guest (VLAN 40) → VLAN 10/20 ping, pre-ACL | Success (confirmed full reachability before lockdown) |
| Guest (VLAN 40) → VLAN 10/20 ping, post-ACL | Failed as expected — 0 replies, `show access-lists` hit counters incrementing on the deny lines |
| Guest DHCP lease renewal, post-ACL | Unaffected — permit rules for UDP 67/68 confirmed working |

## Design Notes / Known Trade-offs

- **ACL isolation is bidirectional in effect, not by explicit design.** The ACL only filters traffic *sourced from* Guest (inbound on `g0/0.40`). Because standard/extended ACLs are stateless, this also blocks Guest's *reply* traffic to any connection another VLAN initiates — so in practice neither side can reach the other, even though only one direction was explicitly filtered. A stateful design (reflexive ACL or zone-based firewall) would be needed to allow VLAN-10-initiated traffic to receive replies from Guest while still blocking Guest-initiated traffic.
- **VLAN 99 subinterface is retained with no hosts on it**, purely to anchor the native VLAN configuration consistently across the trunk and the router. No DHCP pool or ACL was configured for it, since there's currently no traffic to protect.

## Skills Demonstrated
VLAN design and segmentation · 802.1Q trunking · Router-on-a-stick inter-VLAN routing · DHCP server configuration · Extended ACLs for network isolation · Systematic CLI-based verification and troubleshooting
