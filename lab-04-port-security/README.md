# Lab 4 — Port Security and Access Layer Hardening

## Overview
This lab implements Layer 2 access-layer security controls on a Cisco 2960 switch (`SW1`) connected to a gateway router (`R1`) providing DHCP services in Cisco Packet Tracer. It mitigates unauthorized physical access, MAC flooding, rogue switches (BPDU injection), and rogue DHCP servers.

## Objectives
* Isolate active user ports into a dedicated access VLAN (`VLAN 10`) and shut down all unused switch ports into a blackhole VLAN (`VLAN 999`).
* Configure PortFast and BPDU Guard on edge access ports to accelerate legitimate host onboarding while preventing unauthorized switches from altering Spanning Tree topology.
* Enforce Port Security with dynamic Sticky MAC learning, a maximum limit of 1 MAC address per port, and `shutdown` violation enforcement.
* Configure DHCP Snooping on `VLAN 10`, restricting DHCP server offers (`DHCPOFFER`) exclusively to the trusted router uplink (`Gig0/1`).
* Validate defense-in-depth protections through live unauthorized host connection tests and CLI verification.

## Topology
```text
             [ R1 (Gateway & DHCP Server) ]
                 | Gig0/0 (192.168.10.1/24)
                 |
                 | Gig0/1 (Trusted Uplink - VLAN 10)
             [ SW1 (Cisco 2960 Access Switch) ]
              /  \
        Fa0/1/    \Fa0/2   (Fa0/3-24, Gig0/2: Shutdown / VLAN 999)
            /      \
      [ PC1 ]      [ PC2 ]        [ Rogue Laptop ] (Tested on Fa0/1)
  (192.168.10.2) (192.168.10.3)    (192.168.10.50 - Blocked)
```

## VLAN and Addressing Plan

| VLAN ID | Name | Subnet | Default Gateway | Assigned Ports |
| :--- | :--- | :--- | :--- | :--- |
| 10 | USERS | 192.168.10.0/24 | 192.168.10.1 | `Fa0/1`, `Fa0/2`, `Gig0/1` |
| 999 | BLACKHOLE | N/A | N/A | `Fa0/3–24`, `Gig0/2` (Administratively Down) |

## Key Configuration
* **Unused Port Hardening:** All unused interfaces (`Fa0/3–24`, `Gig0/2`) are explicitly set to `switchport mode access`, assigned to `VLAN 999`, and administratively disabled (`shutdown`).
* **Edge STP Hardening:** User ports (`Fa0/1–2`) run `spanning-tree portfast` and `spanning-tree bpduguard enable` to prevent accidental or malicious bridge loops.
* **Port Security (`Fa0/1–2`):** Configured with `switchport port-security`, `maximum 1`, `mac-address sticky`, and `violation shutdown` (places port into `err-disabled` upon unauthorized MAC detection).
* **DHCP Snooping:** Enabled globally and on `VLAN 10` (`ip dhcp snooping vlan 10`), with `no ip dhcp snooping information option` to prevent Option 82 drops on the Cisco router. Only `Gig0/1` is configured with `ip dhcp snooping trust`.

## Verification

| Check | Expected Result |
| :--- | :--- |
| `show vlan brief` | `Fa0/1`, `Fa0/2`, `Gig0/1` in VLAN 10; all unused ports in VLAN 999 |
| `show interfaces status` | Active ports `connected`; all unused ports `disabled` |
| `show ip dhcp snooping` | Enabled on VLAN 10; `Gig0/1` trusted (`yes`), `Fa0/1–2` untrusted (`no`) |
| `show ip dhcp snooping binding` | Dynamic bindings populated for `PC1` (`.2`) and `PC2` (`.3`) |
| `PC1` & `PC2` DHCP & Ping | Valid `192.168.10.x` leases obtained; 0% packet loss to gateway |
| Rogue Laptop plugged into `Fa0/1` | Port transitions to `Secure-shutdown` (`err-disabled`); `Security Violation Count: 1` |

## Design Notes / Known Trade-offs
* **Violation Mode (`shutdown` vs. `restrict`):** `shutdown` was chosen over `restrict` to provide immediate visual alerting and enforce administrative review before re-enabling a compromised wall jack, accepting the trade-off of requiring manual intervention (`shutdown` / `no shutdown`) after a false positive.
* **Single-VLAN Uplink (`Gig0/1`):** Configured as an access port in `VLAN 10` for this single-subnet topology; scaling to multiple user VLANs would require converting `Gig0/1` to an 802.1Q trunk with Router-on-a-Stick subinterfaces while retaining `ip dhcp snooping trust`.

## Skills Demonstrated
Access layer switch hardening, Port Security (Sticky MAC & violation handling), BPDU Guard & PortFast, DHCP Snooping & rogue DHCP mitigation, VLAN segmentation, Cisco IOS CLI troubleshooting.
