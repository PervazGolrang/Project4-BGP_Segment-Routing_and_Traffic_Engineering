# Project 4 - Segment Routing & Traffic Engineering

This repository contains a realistic dual‑site service‑provider topology implemented entirely with **IOS XRv 9000v 24.3.1** images, using Segment Routing MPLS and traffic engineering on CML-2.8.1. The environment demonstrates how a regional ISP (AS 65001) can deliver deterministic traffic engineering, rapid fail‑over, and cost‑optimised peering.

---

## What This Lab Does
This lab builds a regional ISP (AS 65001) operating two Points of Presence (PoPs): **Oslo** and **Bergen**, to learn how real service providers work. Each PoP provides paid transit and public peering for a customer network that is also present in both cities. The lab covers:

- **BGP route reflection** - scaling iBGP without full mesh
- **Segment Routing MPLS** - modern traffic engineering
- **Business relationships** - upstream transit, public peering, customer services
- **Traffic policies** - communities, local preference, prefix filtering
- **Explicit path control** - SR-TE policies for deterministic routing

---

## Network Topology
![`Network Topology with ASN borders`](topology/project4_bgp_sr_te.png)

### Drawio Topology
[`project4_topology.drawio`](topology/project4_bgp_sr_te.drawio)  

---

## Lab Requirements

| Component   | Requirement                       | Notes                                                 |
| ----------- | --------------------------------- | ----------------------------------------------------- |
| RAM         | 64GB with KSM, 180GB without KSM  | 13 routers, ~4GB each                                 |
| vCPU        | 28+ vCPUs                         | 4 vCPU recommended per core node, 1 vCPU per non-core |
| Platform    | CML-2.8.1 or EVE-NG               | IOS XRv 9000v 24.3.1, IOSv images                     |

Refer to [`notes.md`](/notes.md) to tune and enable KSM on CML. Do note it would take up to 15 minutes to fully complete the memory de-duplication.

---

## File Structure

```
├── configs/                    # Device‑specific startup‑configs (.yaml)
├── docs/                       # Network design documentation
│   ├── IP_Plan.md              
│   ├── Design.md               
│   ├── Traffic-engineering.md  
│   ├── Segment-routing.md      
│   └── ASN_Plan.md                             
├── steps/                      # Step-by-step implementation guides
│   ├── 01_Base_BGP_and_IGP.md
│   ├── 02_SR_Enable.md
│   ├── 03_eBGP_Sessions.md
│   ├── 04_BGP_Policies.md
│   └── 05_SR_TE_Policies
├── topology/                   # Network diagrams (.png, .drawio)
├── wireshark/                  # Wireshark packet capture
└── notes.md                    # Lab journal and troubleshooting    
```

## Implementation Steps

**Step 1:** IGP foundation and iBGP route reflectors  
**Step 2:** Enabling SR-MPLS with node-SIDs  
**Step 3:** External BGP sessions (upstream, peers, customers)  
**Step 4:** BGP policies and community frameworks  
**Step 5:** SR-TE policies for traffic engineering

Each step builds on the previous one.

---

## Notes

This lab pushed me way beyond basic networking into real Service Provider territory. The complexity of integrating BGP policies, Segment Routing, and traffic engineering took multiple attempts to get right. Lab journal, troubleshooting, and implementation challenges can be viewed at [`notes.md`](/notes.md).

The goal of this lab was to understand how modern ISPs work, while challenging myself with BGP Segmented Routing and Traffic Engineering.