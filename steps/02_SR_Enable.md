# Step 02 - Segment Routing MPLS

This step activates Segment Routing over MPLS (SR-MPLS) on all core network devices within AS 65001. SR-MPLS provides the foundation for traffic engineering by enabling prefix-based forwarding with globally significant labels and supporting explicit path control through segment lists.

---

## 1. Objective

This phase implements Segment Routing capabilities on the existing MPLS infrastructure:

1. **Global SRGB Configuration:** Establish Segment Routing Global Block (16000-23999) across all core devices
2. **Node-SID Advertisement:** Configure prefix-SIDs for each loopback to enable source routing
3. **SR-MPLS Forwarding:** Populate MPLS tables with segment routing labels for path computation

## SR-MPLS Enabled Devices

| Device         | Role            | SID Index   | Node SID (Label) | SRGB Range    |
| -------------- | --------------- | ----------- | ---------------- |---------------|
| R1_OSLO        | Provider Edge   | 1           | 16001            | 16000-23999   |
| R2_BGO         | Provider Edge   | 2           | 16002            | 16000-23999   |
| RR1_OSLO       | Route Reflector | 11          | 16011            | 16000-23999   |
| RR2_BGO        | Route Reflector | 12          | 16012            | 16000-23999   |
| CORE1_OSLO     | Core Transit    | 21          | 16021            | 16000-23999   |
| CORE2_BGO      | Core Transit    | 22          | 16022            | 16000-23999   |

Upstream ISPs, peering partners, and customer networks do not participate in SR-MPLS. They will receive standard BGP routing without segment routing extensions.

---

## 2. Implementation

### 2.1 Global Segment Routing Configuration

Enable SR-MPLS globally on all core devices:

```bash
segment-routing mpls
!
segment-routing mpls
 global-block 16000 23999
!
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

### 2.2 OSPFv3 Segment Routing Extensions

Enable SR extensions in OSPFv3 for prefix-SID advertisement. 
Note that the process number should match [`Step 01`](/steps/01_Base_BGP_and_IGP.md) configuration:


```bash
router ospfv3 10
 address-family ipv4 unicast
  segment-routing mpls
  area 0 segment-routing mpls
```

### 2.3 Node-SID Configuration

Configure prefix-SID on each device's loopback interface. The explicit-null option preserves QoS markings (MPLS EXP bits) and TTL all the way to the egress router, preventing loss of MPLS metadata that would occur with implicit-null.


| Device      | Interface Configuration                    |
|-------------|--------------------------------------------|
| R1_OSLO     | `ospfv3 prefix-sid index 1 explicit-null`  |
| R2_BGO      | `ospfv3 prefix-sid index 2 explicit-null`  |
| RR1_OSLO    | `ospfv3 prefix-sid index 11 explicit-null` |
| RR2_BGO     | `ospfv3 prefix-sid index 12 explicit-null` |
| CORE1_OSLO  | `ospfv3 prefix-sid index 21 explicit-null` |
| CORE2_BGO   | `ospfv3 prefix-sid index 22 explicit-null` |

Apply to Loopback0 on each device:

```bash
interface Loopback0
 ospfv3 prefix-sid index <index> explicit-null
```

---

## 3.0 Verification and Validation

### 3.1 Segment Routing Global Configuration

Verify SR-MPLS global state and SRGB allocation:

```bash
show segment-routing mpls state
show segment-routing mpls gb
show segment-routing mpls mapping-server
```

SR-MPLS state should show `Enabled`, SRGB range should display `16000-23999`, and there shouldn't be any mapping-server conflicts or errors.

### 3.2 OSPFv3 Segment Routing Database

Confirm prefix-SID advertisement in OSPF topology database:

```bash
show ospfv3 database
show ospfv3 database prefix
show ospfv3 database segment-routing
```

Prefix LSAs should contain SID Index fields for all loopbacks. SID Index values should match configuration (1, 2, 11, 12, 21, 22) with M-bit and NP-bit properly set in prefix-SID sub-TLVs.

### 3.3 MPLS Forwarding Table with SR Labels

Validate SR label installation in MPLS forwarding table:

```bash
show mpls forwarding-table
show mpls forwarding-table labels 16001 16022
show segment-routing mpls lb
```

SR labels 16001-16022 should be installed with the correct next-hops. Expect `Pop Label` action for directly connected node-SIDs and `Swap` operations for remote node-SIDs via the optimal IGP paths.

### 3.4 SR-MPLS Connectivity Testing

Test end-to-end connectivity using SR labels:

```bash
ping mpls ipv4 16021 source 10.255.1.1
ping mpls ipv4 16022 source 10.255.1.2
traceroute mpls ipv4 16021 source 10.255.1.6
```

Expect a 100% success rate for all SR label pings with response times under 10ms in lab environment, confirmed via Wireshark. Traceroute should show label operations (Push/Swap/Pop) correctly.

---

## 4.0 Troubleshooting Common Issues

**SR-MPLS Not Enabled:** If `show segment-routing mpls state` shows "Disabled", verify the `segment-routing mpls` global configuration and check the platform support.

**Prefix-SID Not Advertised:** Missing SID Index in OSPF database would mean that there's a missing `ospfv3 prefix-sid index` configuration on Loopback0 or an incorrect area configuration.

**Label Range Conflicts:** SRGB allocation failures or label binding errors would result from an overlapping label range with LDP, or a static MPLS configurations.

**SR Forwarding Failures:** MPLS ping failures to node-SIDs could mean issues with IGP convergence, MPLS interface enablement, or label operations.

---

## 5.0 Rollback

To disable Segment Routing and revert to pure LDP operation:

```bash
interface Loopback0
 no ospfv3 prefix-sid index <index> explicit-null
!
router ospfv3 10
 address-family ipv4 unicast
  no segment-routing mpls
  no area 0 segment-routing mpls
!
no segment-routing mpls
```