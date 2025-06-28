# Project 4 - BGP Segment‑Routing & Traffic Engineering

## Overview

This repository contains a complete, dual‑site service‑provider topology implemented entirely with **Cat8000v 17.15.01a** images. The environment demonstrates how a regional ISP (AS 65001) can deliver deterministic traffic engineering, rapid fail‑over, and cost‑optimised peering by combining:

* **BGP (IPv4 & IPv6)** - eBGP to upstreams, peers, and customers; iBGP via route reflectors
* **Segment Routing - MPLS (SR‑MPLS)** - prefix/adjacency/peer SIDs, global SRGB 16000‑23999
* **Segment‑Routing Traffic Engineering (SR‑TE)** - colour‑coded policies, candidate‑path preference, and on‑demand nexthop
* **Fast convergence** - BFD at 50 ms, tuned BGP timers, and prefix‑SID based forwarding
* **Policy control** - local‑preference, MED, AS‑path prepending, and community‑based black‑holing

Geographically, the ISP operates two Points of Presence (PoPs): **Oslo** and **Bergen**. Each PoP provides paid transit (single provider, two PoPs), public peering, and dual‑stack connectivity for a customer network that is also present in both cities.

## Topology Highlights

* **Nodes**: 13 Cat8000v routers + 1 TinyCore‑Linux server (for reachability tests)
* **Logical roles**:

  * **EDGE routers**: R1_OSLO, R2_BGO
  * **Route‑Reflectors**: RR1_OSLO, RR2_BGO
  * **Core**: CORE1_OSLO, CORE2_BGO
  * **Upstreams**: ISP1_OSLO, ISP1_BGO (same ASN, different PoPs)
  * **Public Peers**: PEER1_OSLO, PEER2_BGO
  * **Customer Edges**: CUST1_OSLO, CUST2_BGO
  * **Customer Core & Server**: CUST_CORE_OSLO, CUST_SRV1

### Topology

[`project4_bgp_sr_te.drawio`](topology/project4_bgp_sr_te.drawio)  
[`project4_bgp_sr_te.png`](topology/project4_bgp_sr_te.png)

## Repository Structure

```
PROJECT4-BGP_SEGMENT-ROUTING_AND_TRAFFIC_ENGINEERING/
├── configs/                    # Device‑specific startup‑configs (.yaml)
├── docs/
│   ├── IP_Plan.md              # All /30 links and loopback /32s
│   ├── Design.md               # Detailed design narrative (architecture, SRG allocations)
│   ├── Traffic-engineering.md  # Colour map, policy table, binding‑SID catalogue
│   ├── Segment-routing.md      # SRGB ranges, node SID allocations, BGP-LS setup
│   └── Verification.md         # Ping/trace/bfd/convergence test scripts
├── steps/                      # Ordered implementation guides
│   ├── 01_Base_BGP.md
│   ├── 02_SR_Enable.md
│   ├── 03_RR_Setup.md
│   ├── 04_Peering_Policy.md
│   └── 05_SR_TE_Test.md
├── topology/                   # Topologies in visual .png and editable .drawio
│   ├── project4_bgp_sr_te.drawio
│   └── project4_bgp_sr_te.png
└── README.md                   # This file
```

## Prerequisites

| Requirement    | Minimum                              | Notes                                 |
| -------------- | ------------------------------------ | ------------------------------------- |
| Hypervisor RAM | 32 GB                                | 18 routers + 1 Linux container        |
| vCPU count     | 16                                   | Lab stable at 5% CPU‑idle             |
| CML‑2 images   | cat8000v 17.15.01a, alpine‑linux     | EVE‑NG works equivalently             |

## Implementation Milestones

1. **Base BGP & iBGP** - edges peer with RRs; core learns internal routes.
2. **Enable SR‑MPLS** - configure SRGB 16000‑23999, advertise prefix‑SIDs, enable adjacency‑SIDs.
3. **Public Peering & Policy** - inbound / outbound filtering, ORF, NO_EXPORT, MED.
4. **Traffic Engineering** - build colour 100 (Premium) and colour 200 (Default) SR‑TE policies
5. **Fast Failover** - apply BFD at 50 ms on transit links; verify sub‑second convergence.
6. **Blackhole Scenario** - tag /32 host route with community `blackhole:666`; ensure upstream discards it.

## Enhancements

| Enhancement File            | Description                                        |
|-----------------------------|----------------------------------------------------|
| 01_BGP_Failover.md          | BFD and BGP convergence tuning                     |
| 02_Communities_Blackhole.md | Injecting discard routes with communities          |
| 03_IPv6_DualStack.md        | Dual-stack routing using iBGP/eBGP for IPv6        |

## Experimental Modules (WIP)

| Experimental File          | Description                                                           |
|----------------------------|-----------------------------------------------------------------------|
| 01_SRv6_DataPlane.md       | Implement 2001:db8:face::/48 locator with SID re-advertising.         |
| 02_Route_Scale_Testing.md  | BMP replay of 1 million+ routes into ISP1_OSLO for stress validation. |
| 03_Telemetry_Streaming.md  | Export SR-TE stats using gNMI to InfluxDB + Grafana dashboards.       |
