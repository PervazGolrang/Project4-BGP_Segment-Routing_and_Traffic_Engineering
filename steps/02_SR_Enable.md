# Step 02 – Segment Routing MPLS Enablement

This step activates Segment Routing over MPLS (SR-MPLS) on all core network devices within AS 65001. SR-MPLS provides the foundation for traffic engineering by enabling prefix-based forwarding with globally significant labels and supporting explicit path control through segment lists.

---

## 1. Objective

This phase implements Segment Routing capabilities on the existing MPLS infrastructure:

1. **Global SRGB Configuration**: Establish Segment Routing Global Block (16000-23999) across all core devices
2. **Node-SID Advertisement**: Configure prefix-SIDs for each loopback to enable source routing
3. **SR-MPLS Forwarding**: Populate MPLS tables with segment routing labels for path computation

---

## 2. Network Scope

### SR-MPLS Enabled Devices

| Device         | Role            | SID Index   | Node SID (Label) | SRGB Range    |
| -------------- | --------------- | ----------- | ---------------- |---------------|
| **R1_OSLO**    | Provider Edge   | 1           | 16001            | 16000-23999   |
| **R2_BGO**     | Provider Edge   | 2           | 16002            | 16000-23999   |
| **RR1_OSLO**   | Route Reflector | 11          | 16011            | 16000-23999   |
| **RR2_BGO**    | Route Reflector | 12          | 16012            | 16000-23999   |
| **CORE1_OSLO** | Core Transit    | 21          | 16021            | 16000-23999   |
| **CORE2_BGO**  | Core Transit    | 22          | 16022            | 16000-23999   |

### Excluded Devices
Upstream ISPs, peering partners, and customer networks do **not** participate in SR-MPLS. They will receive standard BGP routing without segment routing extensions.

---

## 3. Prerequisites

- **Step 01 Complete**: OSPFv3, LDP, and iBGP sessions established and operational
- **MPLS Foundation**: All core interfaces have `mpls ip` enabled
- **Loopback Reachability**: All /32 loopback prefixes reachable via IGP
- **Label Distribution**: LDP operational with implicit-null and PHP behaviors

---

## 4. Implementation Procedure

### 4.1 Global Segment Routing Configuration

Enable SR-MPLS globally on all core devices.

```bash
# Global SR-MPLS activation
segment-routing mpls
!
# Configure global SRGB range
segment-routing mpls
 global-block 16000 23999
!
# Enable connected prefix-SID advertisement
segment-routing mpls
 connected-prefix-sid-map
  address-family ipv4
   10.255.1.1/32 index 1
   10.255.1.2/32 index 2
   10.255.1.3/32 index 11
   10.255.1.4/32 index 12
   10.255.1.5/32 index 21
   10.255.1.6/32 index 22
```

### 4.2 OSPFv3 Segment Routing Extensions

Enable SR extensions in OSPFv3 for prefix-SID advertisement.

```bash
router ospfv3 10
 address-family ipv4 unicast
  segment-routing mpls
  area 0 segment-routing mpls
```

### 4.3 Node-SID Configuration

| Device      | Interface Configuration                 |
|-------------|-----------------------------------------|
| R1_OSLO     | `ospf prefix-sid index 1 explicit-null` |
| R2_BGO      | `ospf prefix-sid index 2 explicit-null` |
| RR1_OSLO    | `ospf prefix-sid index 3 explicit-null` |
| RR2_BGO     | `ospf prefix-sid index 4 explicit-null` |
| CORE1_OSLO  | `ospf prefix-sid index 5 explicit-null` |
| CORE2_BGO   | `ospf prefix-sid index 6 explicit-null` |

Configure prefix-SID on each device's loopback interface.

```bash
 interface Loopback0
 ospfv3 prefix-sid index [index] explicit-null
```

Change index with the correct SID Index

The reason for using `explicit-null` is to preserve QoS markings (MPLS EXP bits) and TTL all the way to the egress router. If `implicit-null` is continued to be used, the final MPLS label is removed one hop early, and the packet arrives as label 0, losing MPLS metadata.

---

## 5. Verification and Validation

### 5.1 Segment Routing Global Configuration

Verify SR-MPLS global state and SRGB allocation:

```bash
show segment-routing mpls state
show segment-routing mpls gb
show segment-routing mpls mapping-server
```

**Expected Results:**
- SR-MPLS state shows **Enabled**
- SRGB range displays **16000-23999**
- No mapping-server conflicts or errors

### 5.2 OSPFv3 Segment Routing Database

Confirm prefix-SID advertisement in OSPF topology database:

```bash
show ospfv3 database
show ospfv3 database prefix
show ospfv3 database segment-routing
```

**Expected Results:**
- Prefix LSAs contain **SID Index** fields for all loopbacks
- SID Index values match configuration (1-6)
- **M-bit** and **NP-bit** properly set in prefix-SID sub-TLVs

### 5.3 MPLS Forwarding Table with SR Labels

Validate SR label installation in MPLS forwarding table:

```bash
show mpls forwarding-table
show mpls forwarding-table labels 16001 16022
show segment-routing mpls lb
```

**Expected Results:**
- SR labels **16001-16022** installed with correct next-hops
- **Pop Label** action for directly connected node-SIDs
- **Swap** operations for remote node-SIDs via optimal IGP paths

### 5.4 SR-MPLS Connectivity Testing

Test end-to-end connectivity using SR labels:

```bash
ping mpls ipv4 16021 source 10.255.1.1
ping mpls ipv4 16022 source 10.255.1.2
traceroute mpls ipv4 16021 source 10.255.1.6
```

**Expected Results:**
- **100% success rate** for all SR label pings
- **Sub-5ms latency** within lab environment
- Traceroute shows **label operations** (Push/Swap/Pop) correctly

---

## 6. Troubleshooting Common Issues

### SR-MPLS Not Enabled
- **Symptom**: `show segment-routing mpls state` shows **Disabled**
- **Resolution**: Verify `segment-routing mpls` global configuration and check platform support

### Prefix-SID Not Advertised
- **Symptom**: Missing SID Index in OSPF database
- **Resolution**: Confirm `ospf prefix-sid index` on Loopback0 and verify area configuration

### Label Range Conflicts
- **Symptom**: SRGB allocation failures or label binding errors
- **Resolution**: Check for overlapping label ranges with LDP or static MPLS configurations

### SR Forwarding Failures
- **Symptom**: MPLS ping failures to node-SIDs
- **Resolution**: Verify IGP convergence, MPLS interface enablement, and label operations

---

## 7. Rollback Procedure

To disable Segment Routing and revert to pure LDP operation:

```bash
# Remove SR configuration from interfaces
interface Loopback0
 no ospfv3 prefix-sid index [index] explicit-null
!
# Disable SR in OSPFv3
router ospfv3 10
 address-family ipv4 unicast
  no segment-routing mpls
  no area 0 segment-routing mpls
!
# Disable global SR-MPLS
no segment-routing mpls
```

Execute these commands on all six core devices to restore pure LDP-based MPLS operation.