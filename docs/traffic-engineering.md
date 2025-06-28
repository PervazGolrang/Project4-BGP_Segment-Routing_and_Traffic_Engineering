# Traffic Engineering (TE) Policies

This document defines the Segment Routing Traffic Engineering (SR-TE) policy catalog, it includes:

* Color-to-path mapping
* Candidate-path preferences
* Binding-SID assignments
* Explicit SID lists

---

## TE Color Map

| Color | Purpose                  | Description                          |
| ----- | ------------------------ | ------------------------------------ |
| 100   | Low-latency core path    | Via CORE1_OSLO -> CORE2_BGO          |
| 200   | Backup ISP failover path | Via R1_OSLO -> ISP2_BGO -> R2_BGO    |
| 300   | Customer-specific path   | Force through CUST_CORE_OSLO -> CORE |

---

## Binding-SID Allocation

| Policy Name       | Color | Headend     | BSID   | Comments                   |
| ----------------- | ----- | ----------- | ------ | -------------------------- |
| TE_CORE_LOWLAT    | 100   | R1_OSLO     | 16001  | Preferred for core traffic |
| TE_BACKUP_ISP     | 200   | R1_OSLO     | 16002  | Activated on loss of ISP1  |
| TE_CUST1_POLICY   | 300   | CUST1_OSLO  | 16003  | Customer-specific handling |

---

## Policy SID Lists

### TE_CORE_LOWLAT (Color 100)

```bash
Segment List: [CORE1_OSLO (Prefix-SID), CORE2_BGO (Prefix-SID)]
Preference: 200
```

### TE_BACKUP_ISP (Color 200)

```bash
Segment List: [ISP2_BGO (Prefix-SID), R2_BGO (Prefix-SID)]
Preference: 100
```

### TE_CUST1_POLICY (Color 300)

```bash
Segment List: [CUST_CORE_OSLO (Adj-SID), CORE1_OSLO (Prefix-SID)]
Preference: 150
```

---

## Notes

* All policies use **dynamic candidate-paths** unless otherwise defined.
* BGP-LS is assumed to be distributing SIDs and topology.
* Policy activation is monitored using 
    `show segment-routing traffic-eng policy` - Policy status and activaton
    `show segment-routing traffic-eng tunnels`- Tunnel state and path information
    `show segment-routing mpls interfaces`- Interface-level SR-MPLS information