# Lab 3 — Spanning Tree Configuration and Loop Diagnosis

## Overview
This lab builds a physically redundant triangle topology across three switches, configures Rapid PVST+ with deliberate root bridge placement, and then demonstrates — through a live, deliberately-induced failure — exactly what Spanning Tree prevents and what happens when it's removed. Built and verified in Cisco Packet Tracer.

## Objectives
- Build a triangle topology creating a physical Layer 2 loop
- Configure Rapid PVST+ with manually placed primary and secondary root bridges
- Harden PC-facing ports with PortFast and BPDU Guard
- Verify correct root election and identify the automatically blocked port
- Deliberately disable STP to observe and diagnose a real Layer 2 loop
- Restore STP and confirm recovery

## Topology
            SwitchA (Root, priority 4096)
                /            \
               /              \
      SwitchB (priority 8192)  SwitchC (default priority)
               \______________/
                (direct link)

Three switches connected in a full triangle — SwitchA↔SwitchB, SwitchB↔SwitchC, SwitchC↔SwitchA — creating a physical loop. Each switch has one PC attached for connectivity/traffic testing.

## Key Configuration

- **Rapid PVST+** enabled globally on all three switches (`spanning-tree mode rapid-pvst`) instead of default PVST+, for faster convergence.
- **Root bridge placement is deliberate, not left to chance:** SwitchA set to priority 4096 (primary root), SwitchB set to priority 8192 (secondary root/backup), SwitchC left at default (32768).
- **PortFast + BPDU Guard** applied to every PC-facing access port — skips the listening/learning delay for ports where a loop is physically impossible, and immediately disables the port if it ever receives a BPDU (protects against an accidental switch-to-access-port connection).

Full CLI configuration is in [`configs/`](./configs).

## Verification — Normal Operation

| Check | Result |
|-------|--------|
| `show spanning-tree vlan 1` (SwitchA) | Confirms "This bridge is the root" |
| `show spanning-tree vlan 1` (SwitchB, SwitchC) | Root ID matches SwitchA's priority/MAC; each shows one Root Port |
| Port roles across the triangle | Exactly one port in the whole topology sits in `Altn/BLK` (Blocking) — the redundant link between SwitchB and SwitchC, with SwitchC's side blocked and SwitchB's side Designated/Forwarding |
| Cross-switch ping (e.g. PC on SwitchB → PC on SwitchC) | Success — confirms the tree still provides full connectivity despite one physical link being logically unused |

## Loop Diagnosis Exercise

**Attempt 1 — disable STP on a single switch (SwitchA):**
Disabled `no spanning-tree vlan 1` on SwitchA only, expecting the neighboring switches to age out SwitchA's stale root information (RSTP typically detects this within ~3 missed hello intervals) and transition their blocked port to forwarding. After confirming the command applied correctly on SwitchA (`show running-config` and `show spanning-tree vlan 1` both confirmed STP was off there), SwitchB and SwitchC continued to report SwitchA as root, unchanged, well past the expected aging window. **Finding:** Packet Tracer does not reliably simulate BPDU aging when only one switch in a multi-switch topology stops participating — a documented simulator limitation encountered during this lab, not a configuration error.

**Attempt 2 — disable STP on all three switches:**
To produce a reliable, observable loop, `no spanning-tree vlan 1` was applied to all three switches simultaneously, removing all loop-prevention logic from the topology at once. `show spanning-tree vlan 1` on all three confirmed no spanning tree instance existed for VLAN 1. A single test packet was then sent between two PCs on different switches using Packet Tracer's Simulation Mode, and stepped through manually — the packet was observed duplicating and circulating the triangle instead of being delivered once and stopping, visually confirming a Layer 2 broadcast loop.

**Recovery:**
`spanning-tree vlan 1` was re-enabled on all three switches. Root election re-ran automatically; exactly one port returned to Blocking state, restoring the loop-free tree. A repeat of the same Simulation Mode packet test confirmed clean, single delivery with no duplication.

## Design Notes

- **STP doesn't just prevent loops — it removes physical redundancy from the *active* topology while keeping it ready as an instant backup.** The blocked link between SwitchB and SwitchC isn't wasted; if the SwitchA-SwitchB or SwitchA-SwitchC link ever failed, STP would automatically transition that blocked port to forwarding to restore connectivity.
- **Simulator behavior differed from expected real-hardware behavior** in the single-switch STP-disable test. This is documented above as a finding, not glossed over — understanding *why* a lab doesn't behave as expected is itself a diagnostic skill.
- **PortFast/BPDU Guard were intentionally applied only to access ports, never to the inter-switch trunk links** — those links must fully participate in STP, since removing them from consideration is exactly what would allow the kind of loop demonstrated in this lab.

## Skills Demonstrated
Rapid PVST+ configuration · Deliberate root bridge placement · PortFast/BPDU Guard hardening · Root/blocked port verification via CLI · Live loop induction and diagnosis using Simulation Mode · Root-cause documentation of unexpected/simulator behavior
