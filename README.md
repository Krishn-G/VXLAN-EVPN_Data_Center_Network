# VXLAN-EVPN Data Center Network (Cisco NX-OS / CML)

A fully functional Cisco Modeling Labs (CML) lab implementing a **2-Spine / 3-Leaf Clos fabric** using VXLAN with an MP-BGP EVPN control plane. Built on Nexus 9000v (NX-OS 10.4(2)), this lab demonstrates production-grade data center fabric design with a **vPC dual-homed server**, L2 and L3 overlay services, and Symmetric IRB routing — all within a single tenant.

---

## Topology

### Concept Architecture

![Concept Architecture](Topology_Architecture.jpeg)

### CML Canvas

![CML Lab Topology](Topology_CML.png)

**Fabric nodes:** S1, S2 (Spine) — L1, L2 (vPC Leaf Pair), L3 (Standalone Leaf)  
**Hosts:** H0 – H4 (5 endpoints; H1 is dual-homed via vPC)

---

## Node Addressing

| Node | Role              | Loopback0 (Router-ID / VTEP Primary) | Loopback0 Secondary (vPC VIP / VTEP Anycast) | Loopback1 (Anycast-RP)     |
|------|-------------------|---------------------------------------|----------------------------------------------|----------------------------|
| S1   | Spine / RR        | `10.0.0.11/32`                        | —                                            | `10.100.100.100/32`        |
| S2   | Spine / RR        | `10.0.0.12/32`                        | —                                            | `10.100.100.100/32`        |
| L1   | vPC Primary Leaf  | `10.0.0.1/32`                         | `10.1.1.100/32`                              | —                          |
| L2   | vPC Secondary Leaf| `10.0.0.2/32`                         | `10.1.1.100/32`                              | —                          |
| L3   | Standalone Leaf   | `10.0.0.3/32`                         | —                                            | —                          |

> L1 and L2 share the secondary loopback address `10.1.1.100/32` as the vPC Virtual IP (VIP), advertised as the anycast VTEP address for dual-homed traffic.

### Underlay P2P Links (OSPF)

| Link          | Subnet             | S1 Address      | Leaf Address   |
|---------------|--------------------|-----------------|----------------|
| S1 ↔ L1 (E1/1)| `10.1.11.0/24`    | `10.1.11.11`    | `10.1.11.1`    |
| S1 ↔ L2 (E1/2)| `10.2.11.0/24`    | `10.2.11.11`    | `10.2.11.2`    |
| S1 ↔ L3 (E1/3)| `10.3.11.0/24`    | `10.3.11.11`    | `10.3.11.3`    |
| S2 ↔ L1 (E1/1)| `10.1.12.0/24`    | `10.1.12.12`    | `10.1.12.1`    |
| S2 ↔ L2 (E1/2)| `10.2.12.0/24`    | `10.2.12.12`    | `10.2.12.2`    |
| S2 ↔ L3 (E1/2)| `10.3.12.0/24`    | `10.3.12.12`    | `10.3.12.3`    |

---

## vPC Domain (L1 ↔ L2)

L1 and L2 form **vPC Domain 1**, providing active-active dual-homing for H1.

| Parameter              | L1                        | L2                        |
|------------------------|---------------------------|---------------------------|
| vPC Domain ID          | 1                         | 1                         |
| `peer-keepalive` source| `10.255.255.1` (E1/3)     | `10.255.255.2` (E1/3)     |
| `peer-keepalive` dest  | `10.255.255.2`            | `10.255.255.1`            |
| Keepalive VRF          | `VPC`                     | `VPC`                     |
| Peer-Link              | Po100 (E1/4 + E1/5, LACP trunk) | Po100 (E1/4 + E1/5, LACP trunk) |
| Peer-Link VLAN         | VLAN 1234 (`VPC-Peerlink`, infra-vlan) | VLAN 1234 (`VPC-Peerlink`, infra-vlan) |
| vPC member Po10        | E1/7 → H1 (VLAN 10 access)| E1/9 → H1 (VLAN 10 access)|
| `peer-switch`          | ✅                        | ✅                        |
| `peer-gateway`         | ✅                        | ✅                        |
| `ip arp synchronize`   | ✅                        | ✅                        |
| `ipv6 nd synchronize`  | ✅                        | ✅                        |

> `system nve infra-vlans 1234` is configured on both L1 and L2 to exclude VLAN 1234 from VXLAN encapsulation — it is a local vPC infrastructure VLAN only.

---

## Implemented Features

### Underlay

- **OSPF** (`process UNDERLAY`, Area 0) on all P2P links and loopbacks
- **MTU 9216** on all fabric-facing interfaces (required for VXLAN overhead)
- **PIM Sparse-Mode** on all underlay interfaces and loopbacks

### Overlay Control Plane

- **MP-BGP EVPN** (iBGP AS 65000) — Spines S1 and S2 act as **Route Reflectors**
- Route-reflector clients: L1 (`10.0.0.1`), L2 (`10.0.0.2`), L3 (`10.0.0.3`)
- `send-community extended` and `retain route-target all` on both Spines

### BUM Traffic

- **PIM Anycast-RP** at `10.100.100.100` — shared between S1 and S2 via separate Loopback1 interfaces
- Per-VNI multicast groups:

| VNI    | Multicast Group |
|--------|-----------------|
| 100010 | `239.0.0.10`    |
| 100020 | `239.0.0.20`    |

### Overlay Services

| Resource      | Detail                                              |
|---------------|-----------------------------------------------------|
| **VRF**       | `TENANT-A` — L3 VNI 50000 (Symmetric IRB)         |
| **VLAN 10**   | VNI 100010 — on L1, L2                             |
| **VLAN 20**   | VNI 100020 — on L2, L3                             |
| **VLAN 999**  | `L3-TRANSIT` — L3 VNI transit VLAN (all VTEPs)    |
| **VLAN 1234** | `VPC-Peerlink` — vPC infra VLAN (L1, L2 only)     |
| **Anycast GW**| MAC `0000.2222.3333` — identical across all Leafs  |
| **SVI 10**    | `192.168.10.1/24` anycast-gateway (L1, L2)         |
| **SVI 20**    | `192.168.20.1/24` anycast-gateway (L2, L3)         |

---

## Host Connectivity

| Host | IP Address         | Gateway          | Leaf     | Port / Interface        | VLAN | Notes                       |
|------|--------------------|------------------|----------|-------------------------|------|-----------------------------|
| H0   | `192.168.10.10/24` | `192.168.10.1`   | L1       | E1/6                    | 10   | Single-homed                |
| H1   | `192.168.10.11/24` | `192.168.10.1`   | L1 + L2  | Po10 (E1/7 + E1/9)      | 10   | **Dual-homed via vPC**; bond0 (LACP mode 2) |
| H2   | `192.168.10.12/24` | `192.168.10.1`   | L2       | E1/6                    | 10   | Single-homed                |
| H3   | `192.168.20.13/24` | `192.168.20.1`   | L2       | E1/7                    | 20   | Single-homed                |
| H4   | `192.168.20.14/24` | `192.168.20.1`   | L3       | E1/3                    | 20   | Single-homed                |

> H1 runs Linux bonding (`bond0`, mode 2 = balance-xor) with `eth0` connected to L1 E1/7 and `eth1` connected to L2 E1/9, forming vPC Po10 on the switch side.

---

## Design Decisions

**Symmetric IRB** — Inter-subnet routing uses an L3 VNI (VNI 50000, VLAN 999 `L3-TRANSIT`). Routing happens at the ingress VTEP; the remote VTEP only does bridging. This avoids requiring every leaf to carry every tenant's VLAN.

**Route redistribution** — Directly connected routes in `TENANT-A` are redistributed into BGP EVPN via `route-map OVERLAY_SUBNETS` (matching tag 50000), preventing the L3-TRANSIT VLAN route from leaking.

**vPC + VXLAN VTEP anycast** — L1 and L2 share a secondary IP `10.1.1.100/32` on Loopback0 as the vPC VIP. This anycast address is used as the NVE source, so remote VTEPs send encapsulated traffic to a single VTEP IP representing both vPC peers. The NVE source on both is `loopback0`, which carries both the PIP and the shared VIP.

**`peer-switch`** — Both L1 and L2 appear as a single logical switch to connected devices, allowing STP BPDUs from the vPC domain to carry a common bridge MAC.

**`peer-gateway`** — Allows each vPC peer to forward packets destined for the other peer's MAC, preventing traffic black-holing when a host sends traffic to the peer's MAC directly.

**Spine Route Reflectors** — Spines S1 and S2 do not participate in VXLAN data-plane forwarding (no NVE interfaces). They solely reflect EVPN routes between leaf VTEPs.

**`system nve infra-vlans 1234`** — Excludes the vPC peer-link VLAN from VXLAN encapsulation, preventing accidental overlay tunneling of vPC control traffic.

---

## Files in This Repository

| File | Description |
|------|-------------|
| `VXLAN-EVPN Data Center Network.yaml` | CML topology file — import directly into CML to spin up the lab |
| `VXLAN-EVPN Data Center Network.jpeg` | Hand-drawn concept architecture with IP addressing |
| `1782940200931_image.png` | CML canvas screenshot showing the updated topology with vPC |

---

## Getting Started

1. Clone this repository.
2. In CML, go to **Import Lab** and upload `VXLAN-EVPN Data Center Network.yaml`.
3. Start all nodes. Allow 3–5 minutes for NX-OS to boot fully.
4. Verify underlay convergence on any Leaf:
   ```
   show ip ospf neighbor
   show ip route ospf
   ```
5. Verify BGP EVPN sessions (from any Leaf, should show S1 and S2 as Established):
   ```
   show bgp l2vpn evpn summary
   ```
6. Verify the vPC domain on L1 or L2:
   ```
   show vpc
   show vpc peer-keepalive
   show port-channel summary
   ```
7. Verify VXLAN overlay and host reachability:
   ```
   show nve peers
   show nve vni
   show l2route evpn mac all
   show l2route evpn mac-ip all
   ```
8. Test connectivity: ping between H0 ↔ H1 ↔ H2 (same VNI/subnet), then H0 ↔ H3 or H4 (cross-VNI, Symmetric IRB).

### Full Verification Reference

```
# Underlay
show ip ospf neighbor
show ip route ospf
show ip pim neighbor
show ip pim rp

# vPC
show vpc
show vpc peer-keepalive
show port-channel summary
show vpc consistency-parameters global

# EVPN Control Plane
show bgp l2vpn evpn summary
show bgp l2vpn evpn
show l2route evpn mac all
show l2route evpn mac-ip all

# VXLAN Forwarding
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

## Platform

- **Simulator:** Cisco Modeling Labs (CML)
- **Node type:** `nxosv9000`
- **NX-OS version:** 10.4(2)
- **BGP AS:** 65000
- **Host OS:** Alpine Linux (CML built-in)

---

## Roadmap

- [ ] L3 external connectivity via Border Leaf (eBGP peering to upstream router)
- [ ] Additional tenants / VRFs

---

## License

MIT — see [LICENSE](LICENSE) for details.
