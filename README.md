# LANONE Enterprise Edge Network

A multi-layer enterprise network built in EVE-NG (Cisco IOSv), covering access-layer security, Layer 2/3 redundancy, dynamic routing, and a dual-homed Internet edge with NAT and eBGP peering to an upstream provider.

## Topology



| Device | Role | AS / Domain | Loopback0 |
|---|---|---|---|
| Switch13 | Access switch — VLAN 10 | LANONE.com | 10.1.1.4 |
| Switch9 | Access switch — VLAN 20 | LANONE.com | 10.1.1.5 |
| Switch5 | Distribution switch | LANONE.com | 10.1.1.3 |
| vios1 | Edge router 1 | AS 65001 | 10.1.1.1 |
| vios6 | Edge router 2 | AS 65001 | 10.1.1.2 |
| vios7sp | Upstream provider (SP1) | AS 65002 | — (BVI1 203.20.1.3/29) |

## Addressing

| Segment | Subnet | Gateway (HSRP VIP) |
|---|---|---|
| VLAN 10 (Switch13) | 172.168.10.0/24 | 172.168.10.3 |
| VLAN 20 (Switch9) | 172.168.20.0/24 | 172.168.20.3 |
| VLAN 99 (native/mgmt) | 172.168.255.0/24 | — |
| WAN transit (edge ↔ SP1) | 203.20.1.0/29 | — |

vios1 uses `203.20.1.1`, vios6 uses `203.20.1.2`, and vios7sp terminates the block on `203.20.1.3` (BVI1). Loopbacks 10.1.1.1–10.1.1.5 serve as router IDs and BGP/OSPF update sources across the enterprise side.

## Layer 2 design

Switch5 is the distribution point, trunking VLANs 10, 20, and 99 down to Switch9/Switch13 and up to both edge routers over 802.1Q. VLAN 99 is the native VLAN on every trunk. Rapid PVST+ runs throughout, with Switch5 pinned as root for VLANs 10/20/99 (priority 24576) and the access switches set higher (28672) so root placement is deterministic. Switch5 also runs global loopguard to protect against unidirectional link failures on non-designated ports.

Access ports on Switch9 and Switch13 run PortFast with BPDU Guard, and BPDU filtering is enabled globally, keeping STP fast and edge ports isolated from accidental topology changes.

## Layer 2 security

Switch9 (VLAN 20) and Switch13 (VLAN 10) apply an identical hardening profile to every access port:

- **Port security** : max 1500 MAC addresses per port, violation mode `restrict`, 90-minute inactivity aging
- **DHCP snooping** : enabled per-VLAN, with the uplink to Switch5 trusted; all other ports untrusted by default
- **Dynamic ARP Inspection (DAI)** : validates source MAC against the DHCP snooping binding table, rate-limited to 1500 pps per access port, uplink trusted
- **Errdisable recovery** : auto recovers ports disabled by DHCP rate-limit or ARP inspection violations rather than requiring manual intervention

This combination protects against rogue DHCP servers, ARP spoofing/MITM, and MAC flooding at the access layer.

## Layer 3 redundancy and routing

**HSRPv2** pairs vios1 and vios6 as redundant default gateways for each VLAN's SVI, with active-router duties split across both routers for load distribution:

- VLAN 10 (172.168.10.x): vios6 active (priority 255), vios1 standby (254)
- VLAN 20 (172.168.20.x): vios1 active (priority 255), vios6 standby (254)

Both groups use `preempt`, so the configured priority is restored automatically after a failed router returns to service.

**OSPF (area 0)** runs between vios1 and vios6 across the VLAN 10/20/99 sub-interfaces and their loopbacks, providing internal reachability and fast reconvergence for the HSRP VIPs independent of BGP. `default-information originate` injects a default route into OSPF so internal hosts route out through whichever edge router currently holds it, and the SP-facing interface is passive so OSPF adjacencies never form toward the provider.

**iBGP** (AS 65001) peers vios1 and vios6 over their loopbacks with `next-hop-self`, keeping the internal AS's route reflection simple across a two-router edge.

**eBGP** connects vios1 and vios6 (AS 65001) to vios7sp (AS 65002), each session protected with `ttl-security hops 5` to mitigate spoofed/off-path BGP attacks. vios7sp is a lightweight simulated provider edge: it bridges its two Gig interfaces into a single BVI and reaches the enterprise loopbacks via static host routes rather than running an IGP, representing a minimal ISP demarcation point rather than a full provider network.

**NAT/PAT** is configured on both edge routers: internal 172.168.0.0/16 traffic is overloaded onto each router's outside interface toward the SP, backed by a static default route (`0.0.0.0/0` via `203.20.1.3`) so either router can independently provide Internet egress if the other fails.

## Management and hardening

- SSHv2 only, with encryption restricted to AES (128/192/256-CTR)
- `ip http server` / `ip http secure-server` disabled on the edge routers
- Local authentication (`login local`) on console and VTY lines across all devices

## Files

| File | Description |
|---|---|
| `switch5show_running-config` | Distribution switch running-config |
| `switch9_config_do_show_running-c` | Switch9 (VLAN 20 access) running-config |
| `switch13_config_do_show_running-` | Switch13 (VLAN 10 access) running-config |
| `vios1_config_do_show_ip_bgp_summ` | vios1 BGP summary + running-config |
| `vios6_config_do_show_ip_bgp_summ` | vios6 BGP summary + running-config |
| `vios7sp_config_do_show_ip_bgp_su` | vios7sp (SP1) BGP summary + running-config |
| `topology.png` | Topology diagram |
