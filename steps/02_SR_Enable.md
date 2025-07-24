# Step 02 - Segment Routing MPLS

This step activates Segment Routing over MPLS (SR-MPLS) on all core network devices within AS 65001 using **IOS-XR**. SR-MPLS provides the foundation for traffic engineering by enabling prefix-based forwarding with globally significant labels and supporting explicit path control through segment lists.

---

## 1 SR-MPLS Enabled Devices

| Device         | Role            | SID Index   | Node SID (Label) | SRGB Range    |
| -------------- | --------------- | ----------- | ---------------- |---------------|
| R1_OSLO        | Provider Edge   | 1           | 16001            | 16000-23999   |
| R2_BGO         | Provider Edge   | 2           | 16002            | 16000-23999   |
| RR1_OSLO       | Route Reflector | 11          | 16011            | 16000-23999   |
| RR2_BGO        | Route Reflector | 12          | 16012            | 16000-23999   |
| CORE1_OSLO     | Core Transit    | 21          | 16021            | 16000-23999   |
| CORE2_BGO      | Core Transit    | 22          | 16022            | 16000-23999   |

These six network devices will be refered as to the **Core Devices** throughout the project. Upstream ISPs, peering partners, and customer networks do not participate in SR-MPLS. They will receive standard BGP routing without segment routing extensions.

---

## 2. Implementation

### 2.1 Global Segment Routing and Node-SID Configuration

**Per-Device Index Values:**
-    **R1_OSLO**:  `prefix-sid index 1`
-    **R2_BGO**:   `prefix-sid index 2` 
-   **RR1_OSLO**:  `prefix-sid index 11`
-    **RR2_BGO**:  `prefix-sid index 12`
- **CORE1_OSLO**:  `prefix-sid index 21`
-  **CORE2_BGO**:  `prefix-sid index 22`

Enable SR-MPLS globally on all **core** network devices, and configure the prefix-SID on the loopback interface:

```bash
segment-routing
 global-block 16000 23999
 local-block 15000 15999
!
router ospf 10
 segment-routing mpls
 area 0
  mpls traffic-eng
  interface loopback0
    prefix-sid index <INDEX>
```

Since the SRGB-range-config has been changed, the BGP's labels have to be re-programmed as per the new range, commit `process restart bgp`.

---

## 3. Verification and Validation

### 3.1 Segment Routing Global State

Verify that SR-MPLS is enabled and that the SRGB/SRLB ranges are correctly configured:
```bash
show segment-routing
show segment-routing global-block
show segment-routing local-block
```

### 3.2 MPLS Forwarding Table

Validate that SR labels are installed properly:
```bash
show mpls forwarding
show mpls forwarding labels 16001 16022
```

SR labels `16001–16022` should be installed with correct next-hops. **Pop Label** is expected for directly connected node-SIDs, while **Swap** is used for remote node-SIDs along optimal IGP paths.

### 3.3 Node-SID Verification

Confirm that each device's Node-SID is correctly advertised and installed:
```bash
show segment-routing prefix-sid-map ipv4
show ospf database segment-routing detail
```

Each loopback should have its assigned index advertised in OSPF and installed into the MPLS forwarding table.

---

## 4. Troubleshooting Common Issues

**SR not enabled**: Check that `show segment-routing` shows the enabled state and proper SRGB configuration.

**MPLS forwarding missing**: Confirm that MPLS LDP is enabled on the interfaces, and that the SR labels appear in `show mpls forwarding`.

**Label conflicts**: Ensure that the SID indices don't conflict and fall within the SRGB range.

---

## 5. Rollback

```bash
router ospf 10
 no segment-routing mpls
 area 0
  interface Loopback0
   no prefix-sid index
!
config
 no segment-routing
```