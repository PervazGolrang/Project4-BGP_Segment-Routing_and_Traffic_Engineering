# Step 01 - IGP Underlay and BGP Overlay

This step establishes the foundational control-plane infrastructure for the BGP Segment-Routing Traffic Engineering lab. All core network devices participate in a unified OSPFv3 domain to provide IP reachability, while iBGP overlay sessions are established through route reflectors to enable scalable BGP convergence without full-mesh complexity.

---

## 1. Objective

This phase implements the baseline routing infrastructure required before enabling Segment-Routing capabilities:

1. **IP Reachability**: Establish routed connectivity to all /32 loopback addresses across the core network
2. **MPLS Foundation**: Enable MPLS forwarding on all transit interfaces to support future SR-MPLS operations  
3. **iBGP Scalability**: Deploy route reflector hierarchy to avoid n(n-1)/2 iBGP session scaling issues
4. **Label Distribution**: Populate MPLS forwarding tables with implicit-null and PHP behaviors

---

## 2. Core Network Devices

The core network consists of six devices:

| Device      | Role                | IPv4 Loopback | IPv6 Loopback      | Router-ID   | PID |
|-------------|---------------------|---------------|--------------------|-------------|-----|
| R1_OSLO     | Provider Edge       | 10.255.1.1    | 2001:db8:face::1   | 10.255.1.1  | 10  |
| R2_BGO      | Provider Edge       | 10.255.1.2    | 2001:db8:face::2   | 10.255.1.2  | 10  |
| RR1_OSLO    | Route Reflector     | 10.255.1.3    | 2001:db8:face::3   | 10.255.1.3  | 10  |
| RR2_BGO     | Route Reflector     | 10.255.1.4    | 2001:db8:face::4   | 10.255.1.4  | 10  |
| CORE1_OSLO  | Core Transit        | 10.255.1.5    | 2001:db8:face::5   | 10.255.1.5  | 10  |
| CORE2_BGO   | Core Transit        | 10.255.1.6    | 2001:db8:face::6   | 10.255.1.6  | 10  |

---

## 3. Implementation

### 3.1 Base System Configuration

Configure to each core network device.

```bash
hostname <DEVICE_NAME>
!
interface Loopback0
 description "Router-ID"
 ip address <LOOPBACK_IPv4> 255.255.255.255
!
mpls ip
```

### 3.2 Transit Interface Configuration

Configure all point-to-point links between core devices. 
The `point-to-point` network type prevents unnecessary DR/BDR elections on P2P links.


```bash
interface <INTERFACE_ID>
 description "To <Linked-Device>"
 ip address <IPv4_ADDRESS> 255.255.255.254
 ip ospf network point-to-point
!
 mpls ip
!
 ospfv3 10 ipv4 area 0
!
 no shutdown
```

IPv6 configuration is in [`03_IPv6_DualStack.md`](/IPv6_DualStack.md).

### 3.3 OSPFv3 Process Configuration

Configure OSPFv3 with dual-stack capability for future IPv6 implementation:

```bash
router ospfv3 10
!
 address-family ipv4 unicast
  router-id <LOOPBACK_IPv4>
  passive-interface default
  no passive-interface <P2P interfaces>
```

`Passive-interface default` is best practice.

### 3.4 MPLS Label Distribution Protocol

Enable LDP to distribute labels for IGP prefixes and enables PHP behavior. 

**LDP-Enabled Interfaces per Device**

| Device      | LDP-Enabled Interfaces                              |
|-------------|-----------------------------------------------------|
| R1_OSLO     | G4 (RR1), G5 (RR2)                                  |
| R2_BGO      | G4 (RR2), G5 (RR1)                                  |
| RR1_OSLO    | G1 (R1), G2 (CORE1), G3 (R2), G4 (CORE2), G5 (RR2)  |
| RR2_BGO     | G1 (R2), G2 (CORE2), G3 (R1), G4 (CORE1), G5 (RR1)  |
| CORE1_OSLO  | G1 (RR1), G2 (RR2), G4 (CORE2)                      |
| CORE2_BGO   | G2 (RR1), G1 (RR2), G4 (CORE2)                      |

### 3.5 Global LDP Configuration

```bash
mpls ldp router-id <IPv4_LOOPBACK> force
!
interface <INTERFACE_ID>
 mpls ldp igp sync
 mpls ldp discovery transport-address <IPv4_LOOPBACK>     #L0 of device
!
mpls ldp advertise-labels
mpls ldp explicit-null
```

The transport address should match each device's loopback IP (e.g., 10.255.1.1 for R1_OSLO, 10.255.1.4 for RR2_BGO).

### 3.6 BGP Route Reflector Hierarchy

#### Route Reflector Configuration

Route reflector configuration on RR1_OSLO and RR2_BGO:

```bash
router bgp 65001
 bgp router-id <IPv4_RR_LOOPBACK>
 bgp log-neighbor-changes
 bgp cluster-id <IPv4_RR_LOOPBACK>
!
 address-family ipv4 unicast
  neighbor RR-CLIENTS peer-group
  neighbor RR-CLIENTS remote-as 65001
  neighbor RR-CLIENTS update-source Loopback0
  neighbor RR-CLIENTS route-reflector-client
  neighbor RR-CLIENTS send-community both
!
  neighbor 10.255.1.1 peer-group RR-CLIENTS             # R1_OSLO Loopback0
  neighbor 10.255.1.2 peer-group RR-CLIENTS             # R2_BGO Loopback0
  neighbor 10.255.1.5 peer-group RR-CLIENTS             # CORE1_OSLO Loopback0
  neighbor 10.255.1.6 peer-group RR-CLIENTS             # CORE2_BGO Loopback0
!
  neighbor <OTHER_RR_LOOPBACK> remote-as 65001          # E.g. 10.255.1.4 (RR2 L0, if config of RR1)
  neighbor <OTHER_RR_LOOPBACK> update-source Loopback0
  neighbor <OTHER_RR_LOOPBACK> send-community both
!
```

**RR Client Assignments:** Both RR1_OSLO (10.255.1.3) and RR2_BGO (10.255.1.4) serve all four clients: R1_OSLO, R2_BGO, CORE1_OSLO, and CORE2_BGO. This provides full redundancy since all clients connect to both route reflectors.

Route reflector client configuration (applied to R1_OSLO, R2_BGO, CORE1_OSLO, CORE2_BGO):

```bash
router bgp 65001
 bgp router-id <CLIENT_LOOPBACK_IPv4>
 bgp log-neighbor-changes
!
 address-family ipv4 unicast
  neighbor 10.255.1.3 remote-as 65001               # Peer with RR1   
  neighbor 10.255.1.3 update-source Loopback0
  neighbor 10.255.1.3 send-community both
!
  neighbor 10.255.1.4 remote-as 65001               # Peer with RR2
  neighbor 10.255.1.4 update-source Loopback0
  neighbor 10.255.1.4 send-community both
```

---

## 4. Verification and Validation

### 4.1 OSPFv3 Adjacency Verification

Verify that MPLS forwarding entries exist for all loopback prefixes:

```bash
show ospfv3 neighbor
show ospfv3 database
show ip route ospf
```

All neighbor states should show `FULL`. The OSPF database should contain LSAs from all 6 core devices with /32 routes for all loopbacks.

### 4.2 MPLS LDP Validation

```bash
show mpls ldp neighbor
show mpls ldp bindings
show mpls forwarding-table
```
LDP neighbors should show `Oper` state. Label bindings should exist for all /32 loopback prefixes (10.255.1.1-10.255.1.6) with implicit-null for directly connected destinations.

### 4.3 iBGP Session State Verification

Validate BGP session establishment and route reflector functionality:

```bash
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors <neighbor-ip> advertised-routes
```

All BGP sessions should be in `Established` state. Route reflectors should show advertised prefixes to clients, and clients should receive reflected routes from other RR clients.

### 4.4 End-to-End Reachability Testing

Test IP connectivity between all core device loopbacks:

```bash
ping 10.255.1.1 source 10.255.1.6
ping 10.255.1.2 source 10.255.1.5  
traceroute 10.255.1.1 source 10.255.1.6
```

Expect 100% success rate with <10ms response times in lab environment, confirmed with Wireshark. Traceroute paths should follow optimal OSPF cost calculations.

---

## 5. Troubleshooting Common Issues

**BGP Session Failures:** BGP neighbors stuck in Active or Connect state would result to IP reachability issues. Verify `update-source Loopback0` configuration.

**OSPF Adjacency Issues:** Neighbor states showing INIT or 2WAY are due to missing ospf network point-to-point configuration on P2P interfaces.

**MPLS Forwarding Gaps:** Missing LFIB entries mean that the `mpls ip` is not configured on transit interfaces, or OSPF is not advertising loopbacks properly.

**Route Reflector Problems:** RR clients not receiving reflected routes mean there's a missing route-reflector-client configuration or incorrect cluster-id configuration.

---

## 6. Rollback Procedure

To return the network to pre-BGP, pre-MPLS baseline configuration:

```bash
no router bgp 65001
no mpls ldp router-id
no mpls ldp advertise-labels  
no mpls ldp explicit-null
no router ospfv3 10
!
interface <interface-id>
 no mpls ip
 no mpls ldp igp sync
 no mpls ldp discovery transport-address
 no ospfv3 10 ipv4 area 0
 no ospfv3 10 ipv6 area 0
!
no mpls ip
```