# VXLAN-EVPN Data Center Network (Cisco NX-OS / CML)

A fully functional Cisco Modeling Labs (CML) lab implementing a **2-Spine / 3-Leaf Clos fabric** using VXLAN with an MP-BGP EVPN control plane. Built on Nexus 9000v (NX-OS 10.4(2)), this lab demonstrates production-grade data center fabric design with L2 and L3 overlay services across a single tenant.

---

## Topology

### Concept Architecture (Hand-drawn Design)

![Concept Architecture](Topology_Architecture.jpeg)

### CML Canvas (Lab Implementation)

![CML Lab Topology](Topology_CML.png)

**Fabric nodes:** S1, S2 (Spine) — L1, L2, L3 (Leaf)  
**Hosts:** H-0 through H-4 (5 endpoints across 3 leaf switches)

---

## Node Addressing

| Node | Role   | Loopback0 (VTEP/Router-ID) | Loopback1 (Anycast-RP)   |
|------|--------|-----------------------------|---------------------------|
| S1   | Spine  | `10.0.0.11/32`              | `10.100.100.100/32`       |
| S2   | Spine  | `10.0.0.12/32`              | `10.100.100.100/32`       |
| L1   | Leaf   | `10.0.0.1/32`               | —                         |
| L2   | Leaf   | `10.0.0.2/32`               | —                         |
| L3   | Leaf   | `10.0.0.3/32`               | —                         |

### Underlay P2P Links (OSPF)

| Link         | Subnet            |
|--------------|-------------------|
| S1 ↔ L1 (E1/1) | `10.1.11.0/24`  |
| S1 ↔ L2 (E1/2) | `10.2.11.0/24`  |
| S1 ↔ L3 (E1/3) | `10.3.11.0/24`  |
| S2 ↔ L1 (E1/1) | `10.1.12.0/24`  |
| S2 ↔ L2 (E1/2) | `10.2.12.0/24`  |
| S2 ↔ L3 (E1/3) | `10.3.12.0/24`  |

---

## Implemented Features

### Underlay

- **OSPF** (`process UNDERLAY`, Area 0) — all P2P links and loopbacks; MTU 9216 for VXLAN headroom
- **PIM Sparse-Mode** — enabled on all underlay interfaces and loopbacks for BUM traffic replication

### Overlay Control Plane

- **MP-BGP EVPN** (iBGP AS 65000) — Spines act as **Route Reflectors** for all Leaf VTEP peers
- Route-reflector clients: L1 (`10.0.0.1`), L2 (`10.0.0.2`), L3 (`10.0.0.3`)
- `send-community extended` and `retain route-target all` configured on Spines

### BUM Traffic

- **PIM Anycast-RP** at `10.100.100.100` (shared across S1 and S2 via separate Loopback1s)
- Per-VNI multicast groups for BUM replication:
  - VNI 100010 → `239.0.0.10`
  - VNI 100020 → `239.0.0.20`

### Overlay Services

| Service       | Detail                                               |
|---------------|------------------------------------------------------|
| **VRF**       | `TENANT-A` (VNI 50000 — L3 VNI / Symmetric IRB)    |
| **VLAN 10**   | VNI 100010 — hosted on L1 and L2                    |
| **VLAN 20**   | VNI 100020 — hosted on L2 and L3                    |
| **Anycast GW**| `0000.2222.3333` — identical across all Leafs       |
| **SVI 10**    | `192.168.10.1/24` (anycast-gateway on L1, L2)       |
| **SVI 20**    | `192.168.20.1/24` (anycast-gateway on L2, L3)       |

### Host Connectivity

| Host | Leaf | Access Port | VLAN |
|------|------|-------------|------|
| H-0  | L1   | E1/6        | 10   |
| H-1  | L1   | E1/7        | 10   |
| H-2  | L2   | E1/6        | 10   |
| H-3  | L2   | E1/7        | 20   |
| H-4  | L3   | E1/7        | 20   |

All host-facing ports use `spanning-tree port type edge`.

---

## Design Decisions

- **Symmetric IRB** used for inter-subnet routing — L3 VNI (50000) is mapped to VLAN 999 (`L3-TRANSIT`) on all Leafs. Routing happens at the ingress VTEP.
- **Route redistribution** on Leafs: directly connected routes in `TENANT-A` are redistributed into BGP EVPN using `route-map OVERLAY_SUBNETS` (match tag 50000), preventing leaking of transit VLAN routes.
- **`advertise l2vpn evpn`** under `vrf TENANT-A` BGP config ensures Type-5 IP Prefix routes are generated for L3 reachability.
- **Spine Route Reflectors** — Spines do not participate in the overlay forwarding plane (no NVE interfaces); they only reflect EVPN routes between Leaf VTEPs.

---

## Files in This Repository

| File | Description |
|------|-------------|
| `VXLAN-EVPN Data Center Network.yaml` | CML topology file — import directly into CML to spin up the lab |
| `VXLAN-EVPN Data Center Network.jpeg` | Hand-drawn concept architecture with IP addressing |
| `VXLAN-EVPN Data Center Network.png`  | CML canvas screenshot showing node connections |

---

## Getting Started

1. Clone this repository.
2. In CML, go to **Import Lab** and upload `VXLAN-EVPN Data Center Network.yaml`.
3. Start all nodes. Allow 3–5 minutes for NX-OS to boot fully.
4. Verify underlay: `show ip ospf neighbor` on any Leaf should show both Spines.
5. Verify overlay: `show bgp l2vpn evpn summary` on any Leaf should show established sessions to both Spines.
6. Ping between hosts on the same VNI (L2 test), then across VNIs (L3 symmetric IRB test).

### Useful Verification Commands

```
# Underlay
show ip ospf neighbor
show ip route ospf

# PIM / BUM
show ip pim neighbor
show ip pim rp

# EVPN Control Plane
show bgp l2vpn evpn summary
show bgp l2vpn evpn
show l2route evpn mac all
show l2route evpn mac-ip all

# Overlay Forwarding
show nve peers
show nve vni
show vxlan interface
```

---

## Credentials

| Username | Password |
|----------|----------|
| `admin`  | `admin`  |
| `cisco`  | `cisco`  |

---

## Roadmap

- [ ] **vPC Domain** — add dual-homing for hosts via vPC between paired Leaf switches, enabling active-active server connectivity and eliminating STP-blocked uplinks
- [ ] L3 external connectivity (Border Leaf / BGP peering to external router)
- [ ] Additional tenants / VRFs

---

## Platform

- **Simulator:** Cisco Modeling Labs (CML)
- **Node type:** `nxosv9000`
- **NX-OS version:** 10.4(2)
- **BGP AS:** 65000

---

## License

MIT — see [LICENSE](LICENSE) for details.
