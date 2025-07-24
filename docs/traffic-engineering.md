# Traffic Engineering (TE) Policies

This document defines the Segment Routing Traffic Engineering (SR-TE) policy catalog based on the  network topology. It includes color-to-path mapping, candidate-path preferences, binding-SID assignments, and explicit SID lists that correspond to the real connectivity between devices.

---

## TE Color Map

| Color | Purpose                  | Description                               |
| ----- | ------------------------ | ----------------------------------------- |
| 100   | Premium Service          | Low-latency direct paths via shortest RR route |
| 200   | Standard Service         | Best-effort paths via load-balanced RR routes  |
| 300   | Core/Backup Service      | Direct core links and backup via RRs          |

---

## Binding-SID Allocation

| Policy Name        | Color | Headend     | Destination | BSID  | Comments                    |
| ------------------ | ----- | ----------- | ----------- | ----- | --------------------------- |
| OSLO_TO_BGO_PREM   | 100   | R1_OSLO     | 10.255.1.2  | 15100 | Premium via RR2 direct      |
| BGO_TO_OSLO_PREM   | 100   | R2_BGO      | 10.255.1.1  | 15101 | Premium via RR1 direct      |
| OSLO_TO_BGO_STD    | 200   | R1_OSLO     | 10.255.1.2  | 15200 | Standard via both RRs       |
| BGO_TO_OSLO_STD    | 200   | R2_BGO      | 10.255.1.1  | 15201 | Standard via both RRs       |
| CORE_DIRECT        | 300   | CORE1_OSLO  | 10.255.1.6  | 15300 | Direct core-to-core link    |
| CORE_BACKUP        | 300   | CORE2_BGO   | 10.255.1.5  | 15301 | Backup via RR infrastructure |

---

## Policy SID Lists

### Premium Service Policies (Color 100)

**OSLO_TO_BGO_PREM (R1_OSLO -> R2_BGO):**

```bash
- R1_OSLO -> RR2_BGO -> R2_BGO (shortest path)

- R1_OSLO -> RR1_OSLO -> RR2_BGO -> R2_BGO
```

**BGO_TO_OSLO_PREM (R2_BGO -> R1_OSLO):**

```bash
- R2_BGO -> RR1_OSLO -> R1_OSLO (shortest path)

- R2_BGO -> RR2_BGO -> RR1_OSLO -> R1_OSLO
```

### Standard Service Policies (Color 200)

**OSLO_TO_BGO_STD (R1_OSLO -> R2_BGO):**

```bash
- R1_OSLO -> RR1_OSLO -> RR2_BGO -> R2_BGO (load balanced)

- R1_OSLO -> RR2_BGO -> R2_BGO (direct to RR2)
```

**BGO_TO_OSLO_STD (R2_BGO -> R1_OSLO):**

```bash
- R2_BGO -> RR2_BGO -> RR1_OSLO -> R1_OSLO (load balanced)

- R2_BGO -> RR1_OSLO -> R1_OSLO (direct to RR1)
```

### Core Service Policies (Color 300)

**CORE_DIRECT (CORE1_OSLO -> CORE2_BGO):**

```bash
- CORE1_OSLO -> CORE2_BGO (direct link)

- CORE1_OSLO -> RR1_OSLO -> RR2_BGO -> CORE2_BGO (via RR infrastructure)
```

**CORE_BACKUP (CORE2_BGO -> CORE1_OSLO):**

```bash
- CORE2_BGO -> RR2_BGO -> RR1_OSLO -> CORE1_OSLO (backup path via RRs)
```

---

## Node-SID Reference

| Device      | Node-SID | Loopback    | Connectivity                               |
| ----------- | -------- | ----------- | ------------------------------------------ |
| R1_OSLO     | 16001    | 10.255.1.1  | Connects to RR1, RR2, upstreams, customers |
| R2_BGO      | 16002    | 10.255.1.2  | Connects to RR1, RR2, upstreams, customers |
| RR1_OSLO    | 16011    | 10.255.1.3  | Connects to R1, R2, CORE1, CORE2, RR2      |
| RR2_BGO     | 16012    | 10.255.1.4  | Connects to R1, R2, CORE1, CORE2, RR1      |
| CORE1_OSLO  | 16021    | 10.255.1.5  | Connects to RR1, RR2, CORE2, PEER1         |
| CORE2_BGO   | 16022    | 10.255.1.6  | Connects to RR1, RR2, CORE1, PEER2         |

## Traffic Engineering Rationale

**Premium Service (Color 100):** Uses the shortest paths through the RR infrastructure for minimum latency. R1_OSLO uses direct path to RR2_BGO since R2_BGO connects to RR2_BGO locally. Similarly, R2_BGO uses direct path to RR1_OSLO since R1_OSLO connects to RR1_OSLO locally.

**Standard Service (Color 200):** Uses the longer paths through both RRs to provide load balancing and utilize more of the network infrastructure. This distributes traffic more evenly across the backbone links.

**Core Service (Color 300):** CORE1 to CORE2 uses the direct inter-core link for maximum efficiency. The backup policy for CORE2 to CORE1 demonstrates how to use the RR infrastructure when the direct link is unavailable.

---

## Path Selection Logic

The candidate-path preferences are designed to:
1. **Preference 100**: Use optimal paths for each service class
2. **Preference 90**: Provide backup paths with different routing
3. **Preference 30-80**: Dynamic computation fallbacks for resilience

This ensures that the traffic follows the predicted paths under normal conditions, while maintaining automatic failover capabilities during a network failure.