# ASN Plan

This document outlines the Autonomous System Number (ASN) assignments for the BGP Segment Routing and Traffic Engineering lab topology. The lab simulates a regional service provider (AS 65001) operating in two Points of Presence (PoPs): Oslo and Bergen. The ISP provides transit services to customers, purchases upstream connectivity, and participates in settlement-free peering arrangements.

---

## ASN Assignments

| ASN    | Organization Type | Description                         | Devices                                                   |
|--------|-------------------|-------------------------------------|-----------------------------------------------------------|
| 65001  | Regional ISP      | Primary service provider (Our ISP)  | R1_OSLO, R2_BGO, RR1_OSLO, RR2_BGO, CORE1_OSLO, CORE2_BGO |
| 65002  | Upstream ISP      | Transit provider (dual PoP)         | ISP1_OSLO, ISP2_BGO                                       |
| 65003  | Customer          | Multi-site enterprise customer      | CUST1_OSLO, CUST2_BGO, CUST_CORE_OSLO                     |
| 65010  | Peering Partner   | Settlement-free peer (Oslo)         | PEER1_OSLO                                                |
| 65011  | Peering Partner   | Settlement-free peer (Bergen)       | PEER2_BGO                                                 |

---

## BGP Session Types

### iBGP Sessions (Within AS 65001)
All internal routers participate in iBGP using route reflector hierarchy:

**Route Reflector Clients:**
- **RR1_OSLO** serves: R1_OSLO, CORE1_OSLO
- **RR2_BGO** serves: R2_BGO, CORE2_BGO

**Route Reflector Peering:**
- RR1_OSLO ↔ RR2_BGO (iBGP mesh)

### eBGP Sessions

#### Upstream Transit (AS 65002)
- **R1_OSLO** ↔ **ISP1_OSLO** (192.0.2.0/31)
- **R1_OSLO** ↔ **ISP2_BGO** (192.0.2.2/31)
- **R2_BGO** ↔ **ISP1_OSLO** (192.0.2.4/31)
- **R2_BGO** ↔ **ISP2_BGO** (192.0.2.6/31)

#### Public Peering
- **CORE1_OSLO** ↔ **PEER1_OSLO** (192.0.2.26/31)
- **CORE2_BGO** ↔ **PEER2_BGO** (192.0.2.28/31)

#### Customer Connections
- **R1_OSLO** ↔ **CUST1_OSLO** (198.51.100.0/31)
- **R2_BGO** ↔ **CUST2_BGO** (198.51.100.4/31)

---

## Business Relationships

| Relationship Type     | Direction           | ASN Pair          | Policy Notes                                |
|-----------------------|---------------------|-------------------|---------------------------------------------|
| **Customer-Provider** | AS 65001 → AS 65002 | Provider-Upstream | Pay for transit; accept default + specifics |
| **Provider-Customer** | AS 65001 ← AS 65003 | Customer-Provider | Provide transit; announce customer prefixes |
| **Peer-Peer**         | AS 65001 ↔ AS 65010 | Settlement-free   | Exchange customer/own routes only           |
| **Peer-Peer**         | AS 65001 ↔ AS 65011 | Settlement-free   | Exchange customer/own routes only           |

---

## Route Advertisement Policies

### Outbound Announcements

**To Upstream (AS 65002):**
- Own prefixes (192.0.2.0/24, 198.51.100.0/24)
- Customer prefixes (AS 65003 routes)
- Apply AS-path prepending for traffic engineering

**To Customers (AS 65003):**
- Full BGP table (or default route + customer specifics)
- Local-preference manipulation for inbound TE

**To Peers (AS 65010, AS 65011):**
- Own prefixes only
- Customer prefixes only
- NO transit routes (no upstream or other peer routes)

### Inbound Filtering

**From Upstream (AS 65002):**
- Accept all (full table or default)
- Set local-preference based on ingress point

**From Customers (AS 65003):**
- Accept only customer-originated prefixes
- Reject bogons, default routes, and long AS-paths

**From Peers (AS 65010, AS 65011):**
- Accept peer + peer's customer routes
- Reject routes with own ASN in path
- Apply strict prefix filtering 