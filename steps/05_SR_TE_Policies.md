# Step 05 - SR-TE Policies and Traffic Engineering

This step deploys Segment Routing Traffic Engineering (SR-TE) policies using **IOS-XR 9000v** to provide explicit path control, service differentiation, and advanced traffic steering capabilities. The implementation leverages the established SR-MPLS infrastructure and BGP policy framework to enable deterministic routing and service-level optimization.

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

| Color | Service Class | Characteristics             |
|-------|---------------|-----------------------------|
| 100   | Premium       | Low latency, direct paths   |
| 200   | Standard      | Best effort, load balanced  |
| 300   | Core/Backup   | Resilience, alternate paths |

I will not test traffic, the lab will not go that far.

---

## 2. Implementation

SR-TE will be configured on all **core devices**. Two (three in some cases) segment policies, `STD`(Standard) and `PREM` (Premium) will be set up following the **SR-TE color-based policy- and service matrix** above. The `segment-routing traffic-eng` is enabled globally, and logigng is turned on to track policy status.

Then the **segment lists** are defined, that control which labels are used and in what order. These lists are for fixed paths, either altnerative paths, or specific routes via `RR1` and `RR2`. After that, the SR-TE policies are built, where the **standard policy** `OSLO_TO_BGO_STD` uses IGP cost for the dynamic path, and then falls back to explicit paths. The **premium policy** `OSLO_TO_BGO_PREM` prefers the TE metric (better latency), and also includes explicit paths as the backup. Each policy uses color to seperate the service types, and includes multiple preferences for path selection and failover.

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
   autoroute announce
   color 200 end-point ipv4 10.255.1.2
   candidate-paths
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
   autoroute announce
   candidate-paths
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
   autoroute announce
   candidate-paths
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
   autoroute announce
   candidate-paths
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
   autoroute announce
   candidate-paths
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
   autoroute announce
   candidate-paths
    preference 50
     explicit segment-list CORE2_VIA_RR
    !
    preference 100
     explicit segment-list CORE2_DIRECT_PATH
```

## 3. BFD Configuration

BFD will only be configured between **R1_OSLO** and **RR2_BGO**, since this is a simple lab project. The goal is to test that BFD works, and to see how fast failover happens compared to normal OSPF/BGP timers without BFD.

### 3.1 R1_OSLO - Enable BFD on link to RR2_BGO:
```bash
bfd
 interface GigabitEthernet0/0/0/5
  minimum-interval 50
  multiplier 3
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
  minimum-interval 50
  multiplier 3
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

All policies should show `Admin: up`, `Operational: up`. The highest preference candidate-path should be selected.

### 4.2 MPLS Forwarding Verification

Confirm that the SR-TE policies are installed in the data plane:

```bash
show mpls forwarding
show segment-routing traffic-eng forwarding policy
```

Segment lists should show correct label stack operations (Push/Swap/Pop) with valid next-hops.

### 4.3 Manual SR-TE Testing

Manually test traffic over the SR-TE path:

```bash
ping 10.255.1.1 source 10.255.1.6           # CORE2_BGO to R1_OSLO
traceroute 10.255.1.1 source 10.255.1.6     # CORE2_BGO to R1_OSLO
```

- Pinging [`R1_OSLO from CORE1_BGO's Loopback0`](/wireshark/Step05-CORE1_OSLO-to-R1_OSLO-ICMP.pcap)  shows an average ~9ms latency. Route-map stats should show traffic matching. Traceroute should show the expected SR label operations.

### 4.4 Path Verification

Verify explicit paths are working correctly:

```bash
show segment-routing traffic-eng policy detail
```

This confirms which segment-list is being used and that fallback paths are installed as backup.

### 4.5 BFD Verification

Verify that the BFD sessions are up and tied to the OSPF neighbor:
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

Interface `Gi0/0/0/5` on `R1_OSLO` was shut down to simulate failure toward `RR2_BGO`. BFD detected it instantly, and OSPF reconverged in milliseconds. Traffic was rerouted via the backup path through `RR1_OSLO`.

```bash
ping 10.255.1.4      # R1_OSLO to RR2_BGO
```

[`Packet No. 600`](/wireshark/Step05-R1_OSLO-to-RR2_BGO-BFD-showcase.pcap)  shows BFD reporting diagnostic code **0x02** (Echo Function Failure) when the link was shut. A traceroute from [`R1_OSLO`](/wireshark/Step05-R1_OSLO-to-RR1_OSLO-for-RR2_BGO-BFD-traceroute.pcap) confirmed that the first hop was `192.0.2.9` (RR1_OSLO Gi0/0/0/1), and continued through **RR1_OSLO** to [`RR2_BGO`](/wireshark/Step05-RR1_OSLO-to-RR2_BGO-for-R1_OSLO-BFD-traceroute.pcap), confirming the failover path was used as expected.

---

## 5. Troubleshooting

**Policy Not Installing**: Check segment-list reachability, Node-SID advertisement, and binding-SID conflicts.

**Path Selection Issues**: Verify segment-list configurations, Node-SID values, and preference ordering.

**Traffic Not Steering**: Check class-map definitions, policy-map application, and color community advertisement.

**Performance Issues**: Verify explicit paths match expected topology and measure latency with ping/traceroute.

---

## 6. Rollback

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