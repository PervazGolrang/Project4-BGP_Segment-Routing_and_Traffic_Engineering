# Design Document - BGP Segment Routing & Traffic Engineering

## Purpose

This lab demonstrates how a fictional regional ISP (AS 65001) can implement Segment Routing over MPLS (SR-MPLS), traffic engineering with deterministic forwarding, and scalable dual-site interconnectivity using Cat8000v routers. It simulates realistic ISP conditions including public peering, upstreams, and customer edge connectivity.

## High-Level Goals

* Implement scalable BGP-based ISP design with route reflection
* Enable fast reroute (FRR) using SR-MPLS
* Perform deterministic path control with SR-TE
* Introduce realistic peering, customer, and upstream relationships
* Test BGP convergence and policy behavior

## Sites

The lab is split into two primary PoPs:

* **Oslo Site (OSLO)**

  * Devices: R1_OSLO, RR1_OSLO, CORE1_OSLO, ISP1_OSLO, PEER1_OSLO
  * Customers: CUST1_OSLO, CUST_CORE_OSLO, CUST_SRV1_OSLO

* **Bergen Site (BGO)**

  * Devices: R2_BGO, RR2_BGO, CORE2_BGO, ISP2_BGO, PEER2_BGO
  * Customers: CUST2_BGO

## Role Breakdown

| Role             | Device(s)             | Function                                               |
| ---------------- | --------------------- | -------------------------------------------------------|
| Edge Routers     | R1_OSLO, R2_BGO       | Connect to upstream ISPs, RRs, and customers           |
| Route Reflectors | RR1_OSLO, RR2_BGO     | Reflect iBGP routes to internal routers                |
| Core Routers     | CORE1_OSLO, CORE2_BGO | Transit paths, connect to peers and RRs                |
| ISPs             | ISP1_OSLO, ISP2_BGO   | External upstream providers via eBGP                   |
| Peers            | PEER1_OSLO, PEER2_BGO | Public peers for limited route exchange                |
| Customer Edge    | CUST1_OSLO, CUST2_BGO | Simulate BGP customer endpoints using single-home      |
| Customer Core    | CUST_CORE_OSLO        | Mid-layer for internal redistribution at customer site |
| Customer Server  | CUST_SRV1             | TinyCore Linux node for pings and BGP verification     |

## BGP Design

* **ASN 65001**: Regional ISP ASN
* **Route Reflectors**: Dual RR for iBGP scalability (RR1_OSLO and RR2_BGO)
* **eBGP**:
  * ISP1/2 connected via eBGP to R1/R2
  * Customers CUST1/2 use eBGP with private ASNs (e.g. 65010, 65020)
  * PEER1/2 use eBGP with non-transitive sharing (no transit)

## Segment Routing (SR-MPLS)

* **Global SRGB**: 16000–23999
* **Prefix SIDs**: Assigned to loopbacks of all ISP devices
* **Adjacency SIDs**: Used for fast reroute and path pinning
* **Peer SIDs**: For egress SR policy-based steering
* **SID advertisement**: via BGP-LS or SR Extensions to BGP

## SR Traffic Engineering (SR-TE)

* Explicit paths built using:

  * Prefix-SID stacks (e.g. R1 -> R2 -> ISP2)
  * Preference/candidate-path selection
* Used for backup path control and latency-aware path steering

## Policy & Failover

* **BGP Policy Tools**:

  * Local Preference (inbound priority)
  * MED (outbound traffic hint)
  * AS Path Prepending (influence peer path selection)
  * Communities for blackholing (e.g. 65001:666)

* **Failover**:

  * BFD enabled on core and edge router adjacencies
  * Static backup SR-TE policies (fallback)

## Simplifications and Assumptions

* No IGP (OSPF/IS-IS) - BGP used for all routing
* Static SID assignments to simplify TE policy design
* PEER1/PEER2 are stub peers with no internal customers (simulated only)
* Customers are single-homed - no redundancy modeled
* Delay and jitter models not used - CML lab is idealized
* Full-mesh peering between edge, core, and RRs is reflected through BGP RR hierarchy

## Why Cat8000v Only?

* Cat8000v supports modern SR features (SR-MPLS, SR-TE, BFD, gNMI) compared to CSR1000v
* Uniformity across devices avoids feature mismatch and simplifies validation
* High configurability under CML 2.8.1 with low footprint (compared to NCS or ASR images)

## Testing Goals

* Measure path convergence and prefix propagation under failure
* Verify traffic steering through SR-TE policy injection
* Observe control-plane changes via 
  `show bgp`, 
  `show segment-routing`, 
  `show bfd`