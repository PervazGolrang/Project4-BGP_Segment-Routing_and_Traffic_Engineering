# Segment Routing (SR‑MPLS) Design

This document **only** captures the Segment‑Routing (SR‑MPLS) information for the **SR‑participating routers** in the lab. Upstreams, peers, and customer devices do **not** receive SIDs because they are outside the SR domain.

---

## Global Parameters

| Parameter  | Value              |
| ---------- | ------------------ |
| **SRGB**   | 16000 – 23999      |
| Data‑plane | MPLS (PHP enabled) |

This lab uses an SRGB base label of `16000`.
---

## Node (Prefix) SIDs

| Device         | Loopback0  | SID Index   | Node SID (Label) |
| -------------- | ---------- | ----------- | ---------------- |
| **R1_OSLO**    | 10.255.1.1 | 1           | 16001            |
| **R2_BGO**     | 10.255.1.2 | 2           | 16002            |
| **RR1_OSLO**   | 10.255.1.3 | 11          | 16011            |
| **RR2_BGO**    | 10.255.1.4 | 12          | 16012            |
| **CORE1_OSLO** | 10.255.1.5 | 21          | 16021            |
| **CORE2_BGO**  | 10.255.1.6 | 22          | 16022            |

Only these **six routers** participate in SR‑MPLS. The reason for this design choice is to keep the SR domain internal to the backbone infrastructure while excluding customer-facing, ISP, and peer routers from segment routing operations.

The reason it to isolate the segment routing from external BGP relationships and edge complexity, to ensure that SR-MPLS remains focused on backbone traffic engineering. By limiting the participation to core nodes, the SID allocation becomes simpler to manage and the SRGB space stays clean and organized. This design leverages Segment Routing's key benefits of FRR and deterministic pathing to where it provides the most value, i.e. in the high-speed backbone connections between major network sites.

From an operational standpoint, this implementation minimizes risk and complexity while making the overall design easier to maintain and scale as the network grows, even though this lab is a small simulation. The diverse service requirements and external peering relationships are handled by edge routers, without additional overhead of the segment routing configuration.

---

## Adjacency SIDs

Adjacency SIDs are not manually configured in this topology, OSPF automatically allocates dynamic Adj‑SIDs as needed.

If future explicit hop‑by‑hop SR‑TE path control is required, manual Adj‑SIDs can be added between:

* `R1_OSLO` and `CORE1_OSLO`
* `R2_BGO` and `CORE2_BGO`

Only SR‑participating devices support adjacency SIDs. All non‑SR devices are excluded from SID allocation entirely.
