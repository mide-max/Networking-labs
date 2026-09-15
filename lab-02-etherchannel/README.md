# Lab 2 — EtherChannel Link Aggregation

## Overview
This lab bundles multiple physical links between two switches into single logical EtherChannel interfaces, using both LACP (industry-standard) and PAgP (Cisco proprietary) for direct comparison, then proves the bundle survives a physical link failure with no traffic interruption. Built and verified in Cisco Packet Tracer.

## Objectives
- Bundle physical links into Layer 2 EtherChannels using two different negotiation protocols
- Configure trunking on the resulting logical Port-channel interfaces
- Verify successful bundle formation via CLI
- Prove link-level redundancy through a live failure and recovery test, not just static configuration

## Topology
SwitchA                                    SwitchB
  fa0/1 ---- Po1 (LACP, active/active) ---- fa0/1
  fa0/2 ----                            ---- fa0/2
  fa0/3 ---- Po2 (PAgP, desirable/desirable) - fa0/3
  fa0/4 ----                            ---- fa0/4
  
Two switches connected by 4 physical links, split into two independent 2-link EtherChannel bundles: Po1 running LACP and Po2 running PAgP. Each Port-channel interface is configured as an 802.1Q trunk.

## Key Configuration

- **Po1 (LACP):** `fa0/1` and `fa0/2` bundled with `channel-group 1 mode active` on both switches — `active` initiates LACP negotiation rather than waiting passively.
- **Po2 (PAgP):** `fa0/3` and `fa0/4` bundled with `channel-group 2 mode desirable` on both switches — PAgP's equivalent of active-mode negotiation.
- **Trunking:** Both `interface port-channel 1` and `interface port-channel 2` configured as 802.1Q trunks. Configuration applied once at the logical Port-channel level automatically propagates to all bundled physical members.

Full CLI configuration is in [`configs/`](./configs).

## Verification

| Check | Result |
|-------|--------|
| `show etherchannel summary` (baseline) | Po1 and Po2 both show `SU` (Layer2, in use), all 4 physical members show `(P)` — actively bundled |
| `show interfaces port-channel 1` / `2` | Both logical interfaces up/up |
| Baseline ping across trunk (PC1 → PC on SwitchB) | Success, 0% loss |
| `interface fa0/1` → `shutdown` (simulated link failure on Po1) | `show etherchannel summary` shows `Fa0/1(D) Fa0/2(P)` — bundle stays `SU`, still up on the remaining member |
| Ping across trunk during failure | 4/4 replies, 0% loss, avg 2ms — confirmed no traffic interruption while running on a single physical link |
| `no shutdown` on fa0/1 (recovery) | `show etherchannel summary` shows `Fa0/1(P)` again — rejoined the bundle automatically, no reconfiguration needed |

## Design Notes

- **LACP vs. PAgP:** Both protocols achieved the same functional outcome (link aggregation with automatic failover), differing only in negotiation mechanics — LACP is IEEE 802.3ad (open standard, works across vendors), while PAgP is Cisco-proprietary. In a real deployment, LACP would be the default choice unless working exclusively within an all-Cisco environment with a specific reason to use PAgP.
- **Failover is automatic and requires no manual intervention** — the remaining physical member absorbs all traffic the moment a bundled link fails, and a recovered link rejoins the bundle on its own once brought back up, since the `channel-group` membership persists on the interface even while it's administratively down.
- **Bundling requires matching port configuration.** All member ports in a channel-group must share identical speed, duplex, and switchport mode before they can successfully form a bundle — this wasn't tested as a failure case here, but is a known common misconfiguration in real deployments.

## Skills Demonstrated
Link aggregation (LACP and PAgP) · Layer 2 trunking on logical interfaces · Live failure and recovery testing · Understanding of protocol-level differences (open standard vs. vendor-proprietary)
