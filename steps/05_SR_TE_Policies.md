# Step 05 - SR-TE Policies and Traffic Engineering

This step deploys Segment Routing Traffic Engineering (SR-TE) policies using **IOS-XR** to provide explicit path control, service differentiation, and advanced traffic steering capabilities. The implementation leverages the established SR-MPLS infrastructure and BGP policy framework to enable deterministic routing and service-level optimization.

---

## 1. SR-TE color-based policy- and service matrix

| Color | Source Device | Destination   | Service Type    | Path Type       |
|-------|---------------|---------------|-----------------|-----------------|
| 100   | R1_OSLO       | 10.255.1.2    | Premium/Low-Lat | Direct via RR2  |
| 100   | R2_BGO        | 10.255.1.1    | Premium/Low-Lat | Direct via RR1  |
| 200   | R1_OSLO       | 10.255.1.2    | Standard        | Via both RRs    |
| 200   | R2_BGO        | 10.255.1.1    | Standard        | Via both RRs    |
| 300   | CORE1_OSLO    | 10.255.1.6    | Backup Path     | Direct link     |
| 300   | CORE2_BGO     | 10.255.1.5    | Backup Path     | Direct link     |

**Color-Based Service Classes:**

| Color | Service Class | Characteristics             | Use Cases                 | SLA Requirements |
|-------|---------------|-----------------------------|---------------------------|------------------|
| 100   | Premium       | Low latency, direct paths   | Voice, real-time apps     | <5ms, 99.99%     |
| 200   | Standard      | Best effort, load balanced  | Web traffic, email        | <20ms, 99.9%     |
| 300   | Core/Backup   | Resilience, alternate paths | Disaster recovery, backup | <50ms, 99%       |

I will not test traffic, the lab will not go that far.

---

## 2. Implementation

### 2.1 Global SR-TE Configuration (All Core Devices)

Enable SR-TE globally on all core devices:

```bash
segment-routing
 traffic-eng
  logging 
    policy status
```

### 2.2 R1_OSLO Configuration (Premium + Standard Policies)

**Configure explicit segment lists:**

```bash
segment-routing
 traffic-eng
  segment-list OSLO_BGO_ALT
   index 10 mpls label 16012
   index 20 mpls label 16002
  !
  segment-list OSLO_BGO_VIA_RR2
   index 10 mpls label 16012
   index 20 mpls label 16002
  !
  segment-list OSLO_BGO_STANDARD
   index 10 mpls label 16011
   index 20 mpls label 16012
   index 30 mpls label 16002
  !
  segment-list OSLO_BGO_VIA_RR1_RR2
   index 10 mpls label 16011
   index 20 mpls label 16012
   index 30 mpls label 16002
```

**Configure SR-TE policies:**

```bash
segment-routing
 traffic-eng
  policy OSLO_TO_BGO_STD
   color 200 end-point ipv4 10.255.1.2
   candidate-paths
    preference 50
     dynamic
      pcep
      !
      metric
       type igp
      !
     !
    !
    preference 90
     explicit segment-list OSLO_BGO_ALT
     !
    !
    preference 100
     explicit segment-list OSLO_BGO_STANDARD
     !
    !     
   !
  !
  policy OSLO_TO_BGO_PREM
   color 100 end-point ipv4 10.255.1.2
   candidate-paths
    preference 80
     dynamic
      pcep
      !
      metric
       type te
      !
     !
    !
    preference 90
     explicit segment-list OSLO_BGO_VIA_RR1_RR2
     !
    !
    preference 100
     explicit segment-list OSLO_BGO_VIA_RR2
```

### 2.3 R2_BGO Configuration (Premium + Standard Policies)

**Configure explicit segment lists:**

```bash
segment-routing
 traffic-eng
  segment-list BGO_OSLO_ALT
   index 10 mpls label 16011
   index 20 mpls label 16001
  !
  segment-list BGO_OSLO_VIA_RR1
   index 10 mpls label 16011
   index 20 mpls label 16001
  !
  segment-list BGO_OSLO_STANDARD
   index 10 mpls label 16012
   index 20 mpls label 16011
   index 30 mpls label 16001
  !
  segment-list BGO_OSLO_VIA_RR2_RR1
   index 10 mpls label 16012
   index 20 mpls label 16011
   index 30 mpls label 16001
```

**Configure SR-TE policies:**

```bash
segment-routing
 traffic-eng
  policy BGO_TO_OSLO_STD
   color 200 end-point ipv4 10.255.1.1
   candidate-paths
    preference 50
     dynamic
      pcep
      !
      metric
       type igp
      !
     !
    !
    preference 90
     explicit segment-list BGO_OSLO_ALT
     !
    !
    preference 100
     explicit segment-list BGO_OSLO_STANDARD
     !
    !
   !
  !
  policy BGO_TO_OSLO_PREM
   color 100 end-point ipv4 10.255.1.1
   candidate-paths
    preference 80
     dynamic
      pcep
      !
      metric
       type te
      !
     !
    !
    preference 90
     explicit segment-list BGO_OSLO_VIA_RR2_RR1
     !
    !
    preference 100
     explicit segment-list BGO_OSLO_VIA_RR1
```

### 2.4 CORE1_OSLO Configuration (Backup Policy)

**Configure core backup policies:**

```bash
segment-routing
 traffic-eng
  segment-list CORE1_VIA_RR
   index 10 mpls label 16011
   index 20 mpls label 16012
   index 30 mpls label 16022
  !
  segment-list CORE1_DIRECT_PATH
   index 10 mpls label 16022
  !
  policy CORE1_DIRECT
   color 300 end-point ipv4 10.255.1.6
   candidate-paths
    preference 30
     dynamic
      pcep
      !
      metric
       type te
      !
     !
    !     
    preference 50
     explicit segment-list CORE1_VIA_RR
     !
    !
    preference 100
     explicit segment-list CORE1_DIRECT_PATH
```

### 2.5 CORE2_BGO Configuration (Backup Policy)

**Configure core backup policies:**

```bash
segment-routing
 traffic-eng
  segment-list CORE2_VIA_RR
   index 10 mpls label 16012
   index 20 mpls label 16011
   index 30 mpls label 16021
  !
  segment-list CORE2_DIRECT_PATH
   index 10 mpls label 16021
  !
  policy CORE2_DIRECT
   color 300 end-point ipv4 10.255.1.5
   candidate-paths
    preference 30
     dynamic
      pcep
      !
      metric
       type te
      !
     !
     constraints
      disjoint-path group-id 1 type link
     !
    !
    preference 50
     explicit segment-list CORE2_VIA_RR
     !
    !
    preference 100
     explicit segment-list CORE2_DIRECT_PATH
```

## 3. BFD Configuration

BFD will only be configured between R1_OSLO and RR2_BGO, as this is a simple lab project. The intent of this part is to verify BFD functionality and measure failover time compared to standard OSPF/BGP timers without BFD.

### 3.1 R1_OSLO - Enable BFD on link to RR2_BGO:
```bash
bfd
 interface GigabitEthernet0/0/0/5
  multiplier 3
  rx-interval 50000         # Microseconds, results to 50ms
  tx-interval 50000         # Microseconds, results to 50ms
 !
!
router ospf 10
 area 0
  interface GigabitEthernet0/0/0/5
   bfd fast-detect
```

### 3.2 RR2_BGO - Enable BFD on link to R1_OSLO:
```bash
bfd
 interface GigabitEthernet0/0/0/3
  multiplier 3
  rx-interval 50000         # Microseconds, results to 50ms
  tx-interval 50000         # Microseconds, results to 50ms
 !
!
router ospf 10
 area 0
  interface GigabitEthernet0/0/0/3
   bfd fast-detect
```

---

## 4. Verification

### 4.1 SR-TE Policy State

Verify that all SR-TE policies are operational:

```bash
show segment-routing traffic-eng policy
show segment-routing traffic-eng policy color [color]       # e.g. 300 on CORE1_OSLO
```

All policies should show "Admin: `up`, Operational: `up`" status. The highest preference candidate-path should be selected.

### 4.2 MPLS Forwarding Verification

Confirm SR-TE policy installation:

```bash
show mpls forwarding
show segment-routing traffic-eng forwarding policy
```

Segment lists should show proper label stack operations (Push/Swap/Pop) with valid next-hop interfaces and IP addresses.

### 4.3 Manual SR-TE Testing

Test different SR-TE paths manually:

```bash
ping 10.255.1.1 source 10.255.1.6           # CORE2_BGO to R1_OSLO
traceroute 10.255.1.1 source 10.255.1.6     # CORE2_BGO to R1_OSLO
```

- Pinging [`R1_OSLO from CORE1_BGO's Loopback0`](/wireshark/Step05-CORE1_OSLO-to-R1_OSLO-ICMP.pcap) shows an average ~9ms latency between packets. Route-map statistics should show traffic classification matches. Traceroute should show expected SR label operations.

### 4.4 Path Verification

Verify explicit paths are working correctly:

```bash
show segment-routing traffic-eng policy detail
```

### 4.5 BFD Verification

Verify the BFD sessions are up:
```bash
# On R1_OSLO
show bfd session
show ospf neighbor detail
!
# On RR2_BGO  
show bfd session
show ospf neighbor detail
```

### 4.6 BFD Failover Testing

Interface `Gi0/0/0/5` on `R1_OSLO` was shut down to simulate a failure toward `RR2_BGO`. Since BFD was enabled, the failure was instantly detected, and OSPF reconverged in milliseconds compared to seconds. Traffic was rerouted via the backup path through `RR1_OSLO`.

```bash
ping 10.255.1.4      # R1_OSLO to RR2_BGO
```

- [`Packet No. 600`](/wireshark/Step05-R1_OSLO-to-RR2_BGO-BFD-showcase.pcap) in the main **.pcap** file shows that BFD reported a diagnostic code **0x02** (Echo Function Failure) when Gi0/0/0/5 was shutdown.
- A traceroute from [`R1_OSLO`](/wireshark/Step05-R1_OSLO-to-RR1_OSLO-for-RR2_BGO-BFD-traceroute.pcap), showed that the first hop reached `192.0.2.9` (Gi0/0/0/1 on RR1_OSLO).
- The forwarding continued through **RR1_OSLO** to [`RR2_BGO`](/wireshark/Step05-RR1_OSLO-to-RR2_BGO-for-R1_OSLO-BFD-traceroute.pcap), confirming that the traffic was successfully rerouted due to the BFD-triggered failover.

---

## 5. Troubleshooting

**Policy Not Installing**: Check segment-list reachability, Node-SID advertisement, and binding-SID conflicts.

**Path Selection Issues**: Verify segment-list configurations, Node-SID values, and preference ordering.

**Traffic Not Steering**: Check class-map definitions, policy-map application, and color community advertisement.

**Performance Issues**: Verify explicit paths match expected topology and measure latency with ping/traceroute.

---

## 6. Rollback

To remove all SR-TE policies:

### 6.1 R1_OSLO:
```bash
segment-routing
 traffic-eng
  no logging 
   policy status
  !
  no segment-list OSLO_BGO_VIA_RR2
  no segment-list OSLO_BGO_VIA_RR1_RR2
  no segment-list OSLO_BGO_STANDARD
  no segment-list OSLO_BGO_ALT
  no policy OSLO_TO_BGO_PREM
  no policy OSLO_TO_BGO_STD
```

### 6.2 R2_BGO:
```bash
segment-routing
 traffic-eng
  no logging 
   policy status
  !
  no segment-list BGO_OSLO_VIA_RR1
  no segment-list BGO_OSLO_VIA_RR2_RR1
  no segment-list BGO_OSLO_STANDARD
  no segment-list BGO_OSLO_ALT
  no policy BGO_TO_OSLO_PREM
  no policy BGO_TO_OSLO_STD
```

### 6.3 CORE1_OSLO:
```bash
segment-routing
 traffic-eng
  no logging 
   policy status
  !
  no segment-list CORE_DIRECT_PATH
  no segment-list CORE_VIA_RR
  no policy CORE_DIRECT
```

### 6.4 CORE2_BGO:
```bash
segment-routing
 traffic-eng
  no logging 
   policy status
  !
  no segment-list CORE2_BACKUP_PATH
  no policy CORE_BACKUP
```