# Step 05 - SR-TE Policies and Traffic Engineering

This step deploys Segment Routing Traffic Engineering (SR-TE) policies to provide explicit path control, service differentiation, and advanced traffic steering capabilities. The implementation leverages the established SR-MPLS infrastructure and BGP policy framework to enable deterministic routing and service-level optimization.

---

## 1. Objective

This phase implements advanced traffic engineering capabilities using Segment Routing:

1. **Explicit Path Control:** Deploy SR-TE policies for deterministic routing independence of IGP shortest paths
2. **Service Differentiation:** Implement color-based traffic steering for different service classes
3. **Path Diversity:** Establish primary and backup paths with automatic failover capabilities
4. **Performance Optimization:** Configure latency-optimized and bandwidth-optimized paths
5. **Operational Visibility:** Enable comprehensive monitoring and troubleshooting of SR-TE policies

---

## 2. SR-TE policy matrix

| Policy Name        | Color | Source Device | Destination   | Service Type    | Path Type       | Binding SID |
|--------------------|-------|---------------|---------------|-----------------|-----------------|-------------|
| OSLO_TO_BGO_PREM   | 100   | R1_OSLO       | 10.255.1.2    | Premium/Low-Lat | Direct via RR2  | 15100       |
| BGO_TO_OSLO_PREM   | 100   | R2_BGO        | 10.255.1.1    | Premium/Low-Lat | Direct via RR1  | 15101       |
| OSLO_TO_BGO_STD    | 200   | R1_OSLO       | 10.255.1.2    | Standard        | Via both RRs    | 15200       |
| BGO_TO_OSLO_STD    | 200   | R2_BGO        | 10.255.1.1    | Standard        | Via both RRs    | 15201       |
| CORE_BACKUP_OSLO   | 300   | CORE1_OSLO    | 10.255.1.6    | Backup Path     | Direct link     | 15300       |
| CORE_BACKUP_BGO    | 300   | CORE2_BGO     | 10.255.1.5    | Backup Path     | Direct link     | 15301       |

**Color-based service classes:**

| Color | Service Class | Characteristics              | Use Cases                    | SLA Requirements |
|-------|---------------|------------------------------|------------------------------|------------------|
| 100   | Premium       | Low latency, direct paths    | Voice, real-time apps        | <5ms, 99.99%     |
| 200   | Standard      | Best effort, load balanced   | Web traffic, email           | <20ms, 99.9%     |
| 300   | Core/Backup   | Resilience, alternate paths  | Disaster recovery, backup    | <50ms, 99%       |

---

## 3. Implementation

### 3.1 Global SR-TE Configuration

Enable SR-TE globally and configure path calculation settings:

```bash
segment-routing
 traffic-eng
  logging policy status
  binding-sid range 15000 15999
  auto-tunnel backup
   tunnel-id min 1000 max 1999
```

### 3.2 Premium Service SR-TE Policies (Color 100)

**R1_OSLO premium policy to R2_BGO (shortest path via RR2):**

```bash
segment-routing
 traffic-eng
  policy OSLO_TO_BGO_PREM
   description "Premium low-latency path OSLO to BGO"
   color 100 end-point ipv4 10.255.1.2
   !
   candidate-paths
    preference 100
     description "Direct path via RR2_BGO"
     explicit segment-list name OSLO_BGO_VIA_RR2
      index 10 mpls label 16012        # RR2_BGO
      index 20 mpls label 16002        # R2_BGO
     !
    preference 90
     description "Backup path via RR1_OSLO then RR2_BGO"
     explicit segment-list name OSLO_BGO_VIA_RR1_RR2
      index 10 mpls label 16011        # RR1_OSLO
      index 20 mpls label 16012        # RR2_BGO
      index 30 mpls label 16002        # R2_BGO
     !
    preference 80
     description "Dynamic path computation fallback"
     dynamic
      metric type te
      constraints
       bandwidth 100000
       hop-limit 4
   !
   binding-sid mpls 15100
```

**R2_BGO premium policy to R1_OSLO (shortest path via RR1):**

```bash
segment-routing
 traffic-eng
  policy BGO_TO_OSLO_PREM
   description "Premium low-latency path BGO to OSLO"
   color 100 end-point ipv4 10.255.1.1
   !
   candidate-paths
    preference 100
     description "Direct path via RR1_OSLO"
     explicit segment-list name BGO_OSLO_VIA_RR1
      index 10 mpls label 16011        # RR1_OSLO
      index 20 mpls label 16001        # R1_OSLO
     !
    preference 90
     description "Backup path via RR2_BGO then RR1_OSLO"
     explicit segment-list name BGO_OSLO_VIA_RR2_RR1
      index 10 mpls label 16012        # RR2_BGO
      index 20 mpls label 16011        # RR1_OSLO
      index 30 mpls label 16001        # R1_OSLO
   !
   binding-sid mpls 15101
```

### 3.3 Standard Service SR-TE Policies (Color 200)

**R1_OSLO standard policy configuration (load balanced via both RRs):**

```bash
segment-routing
 traffic-eng
  policy OSLO_TO_BGO_STD
   description "Standard best-effort path OSLO to BGO"
   color 200 end-point ipv4 10.255.1.2
   !
   candidate-paths
    preference 100
     description "Primary path via RR1_OSLO then RR2_BGO"
     explicit segment-list name OSLO_BGO_STANDARD
      index 10 mpls label 16011        # RR1_OSLO
      index 20 mpls label 16012        # RR2_BGO
      index 30 mpls label 16002        # R2_BGO
     !
    preference 90
     description "Alternate path via RR2_BGO direct"
     explicit segment-list name OSLO_BGO_ALT
      index 10 mpls label 16012        # RR2_BGO
      index 20 mpls label 16002        # R2_BGO
     !
    preference 50
     description "Dynamic fallback"
     dynamic
      metric type igp
      constraints
       hop-limit 6
   !
   binding-sid mpls 15200
```

**R2_BGO standard policy configuration:**
```bash
segment-routing
 traffic-eng
  policy BGO_TO_OSLO_STD
   description "Standard best-effort path BGO to OSLO"
   color 200 end-point ipv4 10.255.1.1
   !
   candidate-paths
    preference 100
     description "Primary path via RR2_BGO then RR1_OSLO"
     explicit segment-list name BGO_OSLO_STANDARD
      index 10 mpls label 16012        # RR2_BGO
      index 20 mpls label 16011        # RR1_OSLO
      index 30 mpls label 16001        # R1_OSLO
     !
    preference 90
     description "Alternate path via RR1_OSLO direct"
     explicit segment-list name BGO_OSLO_ALT
      index 10 mpls label 16011        # RR1_OSLO
      index 20 mpls label 16001        # R1_OSLO
   !
```

### 3.4 Backup Path SR-TE Policies (Color 300)

**CORE1_OSLO to CORE2_BGO (direct link):**

```bash
segment-routing
 traffic-eng
  policy CORE_DIRECT
   description "Direct core-to-core path"
   color 300 end-point ipv4 10.255.1.6
   !
   candidate-paths
    preference 100
     description "Direct link CORE1 to CORE2"
     explicit segment-list name CORE_DIRECT_PATH
      index 10 mpls label 16022        # CORE2_BGO
     !
    preference 50
     description "Emergency backup via RRs"
     explicit segment-list name CORE_VIA_RR
      index 10 mpls label 16011        # RR1_OSLO
      index 20 mpls label 16012        # RR2_BGO
      index 30 mpls label 16022        # CORE2_BGO
   !
   binding-sid mpls 15300
```

**CORE2_BGO backup policy to CORE1_OSLO:**
```bash
segment-routing
 traffic-eng
  policy CORE_BACKUP
   description "Backup core path via RR infrastructure"
   color 300 end-point ipv4 10.255.1.5
   !
   candidate-paths
    preference 100
     description "Backup path via RR2 then RR1"
     explicit segment-list name CORE2_BACKUP_PATH
      index 10 mpls label 16012        # RR2_BGO
      index 20 mpls label 16011        # RR1_OSLO
      index 30 mpls label 16021        # CORE1_OSLO
     !
    preference 50
     description "Emergency dynamic path"
     dynamic
      metric type te
      constraints
       hop-limit 8
       disjoint-path type link
   !
   binding-sid mpls 15301
```

### 3.5 Traffic Steering Configuration

Configure traffic steering on the PE routers where customers connect.

**R1_OSLO customer traffic steering (connected to CUST1_OSLO on G1):**

```bash
ip access-list extended VOICE_TRAFFIC
 permit udp any any range 16384 32767
 permit tcp any any eq 1720
 permit tcp any any range 5060 5061
!
ip access-list extended WEB_TRAFFIC  
 permit tcp any any eq 80
 permit tcp any any eq 443
 permit tcp any any eq 8080
!
route-map STEER_TO_PREMIUM permit 10
 match ip address VOICE_TRAFFIC
 set ip next-hop 198.51.100.100
!
route-map STEER_TO_STANDARD permit 20
 match ip address WEB_TRAFFIC
 set ip next-hop 198.51.100.200
!
interface GigabitEthernet1
 description "Connection to CUST1_OSLO"
 ip policy route-map STEER_TO_PREMIUM
```

**R2_BGO customer traffic steering (connected to CUST2_BGO on G1):**

```bash
ip access-list extended VOICE_TRAFFIC
 permit udp any any range 16384 32767
 permit tcp any any eq 1720
 permit tcp any any range 5060 5061

ip access-list extended WEB_TRAFFIC  
 permit tcp any any eq 80
 permit tcp any any eq 443
 permit tcp any any eq 8080

route-map STEER_TO_PREMIUM permit 10
 match ip address VOICE_TRAFFIC
 set ip next-hop 198.51.100.110

route-map STEER_TO_STANDARD permit 20
 match ip address WEB_TRAFFIC
 set ip next-hop 198.51.100.210

interface GigabitEthernet1
 description "Connection to CUST2_BGO"
 ip policy route-map STEER_TO_PREMIUM
```

### 3.6 BGP Color Extended Community Configuration

**R1_OSLO BGP color configuration:**

```bash
ip access-list extended PREMIUM_SERVICES
 permit udp any any range 16384 32767
 permit tcp any any eq 1720
 permit tcp any any range 5060 5061

route-map SET_COLOR permit 10
 description "Premium color for priority services"
 match ip address PREMIUM_SERVICES
 set extcommunity color 100

route-map SET_COLOR permit 20  
 description "Standard color for regular traffic"
 set extcommunity color 200

router bgp 65001
 address-family ipv4 unicast
  neighbor 198.51.100.0 send-community extended
  neighbor 198.51.100.0 route-map SET_COLOR out
```

**R2_BGO BGP color configuration:**

```bash
ip access-list extended PREMIUM_SERVICES
 permit udp any any range 16384 32767
 permit tcp any any eq 1720
 permit tcp any any range 5060 5061

route-map SET_COLOR permit 10
 description "Premium color for priority services"
 match ip address PREMIUM_SERVICES
 set extcommunity color 100

route-map SET_COLOR permit 20  
 description "Standard color for regular traffic"
 set extcommunity color 200

router bgp 65001
 address-family ipv4 unicast
  neighbor 198.51.100.4 send-community extended
  neighbor 198.51.100.4 route-map SET_COLOR out
```

### 3.7 SR-TE Policy Monitoring

Configure basic monitoring and logging:

```bash
segment-routing
 traffic-eng
  logging policy status
  logging policy bsid

performance-measurement
 delay-measurement
  advertise-delay 10
 loss-measurement
  advertise-loss 10
```

---

## 4. Verification

### 4.1 SR-TE Policy State

Verify that all SR-TE policies are operational and active:

```bash
show segment-routing traffic-eng policy all
show segment-routing traffic-eng policy name OSLO_TO_BGO_PREM
show segment-routing traffic-eng policy color 100
show segment-routing traffic-eng binding-sid
```

All policies should show "Admin: `up`, Operational: `up`" status. The highest preference candidate-path should be selected. Unique binding-SIDs should be allocated for each policy in the 15100-15999 range with packet and byte counters incrementing for active policies.

### 4.2 MPLS Forwarding Table

Confirm the SR-TE policy installation in the MPLS forwarding plane:

```bash
show mpls forwarding-table labels 15100 15999
show mpls forwarding-table detail
show mpls traffic-eng tunnels brief
```

Binding-SID entries for labels 15100-15999 should be installed with the correct outgoing actions. Segment lists should show proper label stack operations (Push/Swap/Pop) with valid next-hop interfaces and IP addresses.

### 4.3 Traffic Steering Validation

Testing traffic steering through different SR-TE policies:

```bash
ping 10.255.1.2 source 10.255.1.1 df-bit size 1400
traceroute 10.255.1.2 source 10.255.1.1
debug ip policy
show route-map STEER_TO_PREMIUM
show ip policy interface GigabitEthernet1
```

Premium traffic should route via shortest RR path. Standard traffic should route via load-balanced RR paths. Route-map statistics should show traffic classification matches. Traceroute should show expected SR label operations.

### 4.4 Performance Measurement

Verify the performance measurement and SLA compliance:

```bash
show performance-measurement summary
show performance-measurement interfaces detail
show segment-routing traffic-eng policy all performance-measurement
```

One-way and round-trip delay should be within SLA thresholds. Premium paths should be <5ms, standard paths <20ms, confirmed via Wireshark.

## 4. Troubleshooting Common Issues

**SR-TE Policy Not Installing:** The SR-TE Policy is showing operational state `down` would result from segment-list reachability issues, missing node-SID advertisement, or binding-SID allocation conflicts.

**Binding-SID Allocation Failures:** If the SR-TE Policy is unable to allocate the binding-SID then it could mean binding-SID range conflicts, insufficient available label space, or conflicts with other protocols.

---

## 5. Rollback

To remove all SR-TE policies and revert to IGP-based forwarding:

```bash
no segment-routing traffic-eng policy OSLO_TO_BGO_PREM
no segment-routing traffic-eng policy BGO_TO_OSLO_PREM
no segment-routing traffic-eng policy OSLO_TO_BGO_STD
no segment-routing traffic-eng policy BGO_TO_OSLO_STD
no segment-routing traffic-eng policy CORE_DIRECT
no segment-routing traffic-eng policy CORE_BACKUP
!
interface GigabitEthernet1
 no ip policy route-map STEER_TO_PREMIUM
!
no route-map STEER_TO_PREMIUM
no route-map SET_COLOR
no ip access-list extended VOICE_TRAFFIC
no ip access-list extended WEB_TRAFFIC
!
no segment-routing traffic-eng
no performance-measurement
```