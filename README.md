# Cisco OSPF Single-Area Lab

## Overview

This lab demonstrates a **single-area OSPFv2** network between two Cisco routers. OSPF is a **link-state routing protocol** that offers:

- **Fast convergence** — routers quickly detect topology changes and re-converge using LSAs and SPF.
- **Efficient bandwidth usage** — routes are exchanged only when changes occur, with **triggered, incremental updates** (no periodic full-table routing updates like RIP) and support for **autosummarization-free, classless (VLSM/CIDR)** design.
- **Scalability** — area-based hierarchical design (this lab uses the backbone **Area 0**).

## Topology

```
                        Area 0 (Backbone)

  [PC1]                                                  [PC2]
    |                                                      |
    | (LAN A: 10.0.0.0/24)                                 | (LAN B: 192.168.20.0/24)
    |                                                      |
  +-----+        Serial/      +-----+
  | R1  |----------------------| R2  |
  +-----+   10.0.0.0/30        +-----+
   .1  |         (link)         |  .2
 Router-ID: 1.1.1.1             Router-ID: 2.2.2.2
```

- **R1** and **R2** are connected by the WAN link `10.0.0.0/30` (Area 0).
- Each router serves one local LAN: R1 → `10.0.0.0/24`, R2 → `192.168.20.0/24`.
- PCs use their local router as the default gateway.

## IP Addressing Table

| Device | Interface        | IP Address      | Subnet Mask / Wildcard | Role                        |
|--------|------------------|-----------------|------------------------|-----------------------------|
| R1     | WAN (to R2)      | 10.0.0.1        | 255.255.255.252 / 0.0.0.3 | OSPF link to R2 (Area 0) |
| R1     | LAN interface    | 10.0.0.1        | 255.255.255.0 / 0.0.0.255 | Gateway for LAN A (10.0.0.0/24) |
| PC1    | NIC              | 10.0.0.10       | 255.255.255.0          | Host in LAN A               |
| R2     | WAN (to R1)      | 10.0.0.2        | 255.255.255.252 / 0.0.0.3 | OSPF link to R1 (Area 0) |
| R2     | LAN interface    | 192.168.20.1    | 255.255.255.0 / 0.0.0.255 | Gateway for LAN B (192.168.20.0/24) |
| PC2    | NIC              | 192.168.20.10   | 255.255.255.0          | Host in LAN B               |

## Prerequisites

- **Cisco Packet Tracer** (or a real Cisco/Generic router environment such as GNS3/EVE-NG).
- Basic familiarity with Cisco IOS CLI.
- Two routers, two switches, and two end devices (PCs) to build the topology above.

## Router Configurations

### R1

```
enable
configure terminal

hostname R1

interface GigabitEthernet0/0
 description Link-to-R2
 ip address 10.0.0.1 255.255.255.252
 no shutdown

interface GigabitEthernet0/1
 description LAN-A
 ip address 10.0.0.1 255.255.255.0
 no shutdown

router ospf 1
 router-id 1.1.1.1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.0.0 0.0.0.255 area 0

end
write memory
```

### R2

```
enable
configure terminal

hostname R2

interface GigabitEthernet0/0
 description Link-to-R1
 ip address 10.0.0.2 255.255.255.252
 no shutdown

interface GigabitEthernet0/1
 description LAN-B
 ip address 192.168.20.1 255.255.255.0
 no shutdown

router ospf 1
 router-id 2.2.2.2
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.20.0 0.0.0.255 area 0

end
write memory
```

## Verification Commands

| Command | What It Shows | Expected Result |
|---------|---------------|-----------------|
| `show ip ospf neighbor` | OSPF neighbor state | R1 ↔ R2 in `FULL` state, Router-ID 1.1.1.1 / 2.2.2.2 |
| `show ip route ospf` | OSPF-learned routes | `O 192.168.20.0/24` on R1 and `O 10.0.0.0/24` on R2 (LANs) |
| `show ip ospf interface` | OSPF interface details | Correct area (0), network type, cost, hello/dead timers |
| `show ip protocols` | Active routing protocol info | OSPF process 1, RouterF process 1, Router-ID, advertised networks, area 0 |

Test endPC1**, `ping 192.168.20.10` (PC2) — it should succeed.

## Troubleshooting

| Symptom / Problem | Likely Cause | Solution |
|-------------------|--------------|----------|
| Neighbor stuck in `EXSTART`/`2-WAY` or MTU-related errors | **Subnet mask mismatch** on the shared link | Ensure both ends use the same mask (e.g., `/30`) on the WAN link |
| `%OSPF-4-DUP_RTRID` warning / adjacency fails | **Duplicate Router-ID** in the network | Assign unique Router-IDs (e.g., 1.1.1.1 and 2.2.2.2), then `clear ip ospf process` |
| Neighbor stuck in `ATTEMPT`/`EXCHANGE`, no adjacency | **Area mismatch** between interfaces | Both interfaces must be in the same area (Area 0) |
| Routes missing from routing table | Wrong wildcard mask or missing `network` statement | Verify `network <subnet> <wildcard> area 0` covers each interface |
| Interface not participating in OSPF | Interface is shutdown or IP not configured | Check `no shutdown` and correct IP addressing |
| Intermittent neighbor flaps | Hello/dead timer mismatch or unstable link | Match hello/dead timers; check physical cabling/clock settings |

## Security Notes & Pro-Tips

- **Hardening OSPF:** Enable MD5/SHA authentication per interface (`ip ospf message-digest-key 1 md5 <key>` or `ip ospf authentication`) to prevent rogue neighbor adjacencies.
- **Passive interfaces:** Use `passive-interface GigabitEthernet0/1` for LAN interfaces so OSPF does not send hellos toward end hosts.
- **Switch ports to PCs:** Enable **PortFast** (`spanning-tree portfast`) on access ports to speed up host connectivity and avoid STP delays.
- **Access ports:** Explicitly set `switchport mode access` and match **speed/duplex** (or leave autonegotiation consistent on both ends) to avoid duplex-mismatch issues.
- **Documentation:** Keep the addressing table up to date; use `router-id` explicitly rather than relying on the highest loopback/interface IP.
