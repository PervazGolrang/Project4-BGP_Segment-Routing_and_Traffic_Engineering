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

## 2. Network Scope

### Core Network Devices

| Device      | Role                | IPv4 Loopback | IPv6 Loopback      | Router-ID   | PID |
|-------------|---------------------|---------------|--------------------|-------------|-----|
| R1_OSLO     | Provider Edge       | 10.255.1.1    | 2001:db8:face::1   | 10.255.1.1  | 10  |
| R2_BGO      | Provider Edge       | 10.255.1.2    | 2001:db8:face::2   | 10.255.1.2  | 10  |
| RR1_OSLO    | Route Reflector     | 10.255.1.3    | 2001:db8:face::3   | 10.255.1.3  | 10  |
| RR2_BGO     | Route Reflector     | 10.255.1.4    | 2001:db8:face::4   | 10.255.1.4  | 10  |
| CORE1_OSLO  | Core Transit        | 10.255.1.5    | 2001:db8:face::5   | 10.255.1.5  | 10  |
| CORE2_BGO   | Core Transit        | 10.255.1.6    | 2001:db8:face::6   | 10.255.1.6  | 10  |

### Excluded Devices
Upstream ISPs, peering partners, and customer edge devices are **not** configured in this phase. They will be addressed in subsequent steps focusing on eBGP policy implementation.

---

## 3. Prerequisites

- **Platform**: Cisco Cat8000v running IOS-XE 17.15.01a
- **Physical Connectivity**: All core-to-core links cabled per topology diagram
- **IP Addressing**: Point-to-point links configured per [`IP_Plan.md`](/docs/IP_Plan.md)
- **Base Configuration**: Hostnames and loopback interfaces created on all devices

---

## 4. Implementation Procedure

### 4.1 Base System Configuration Template

Configure the template to each core network device.

```bash
hostname <DEVICE_NAME>
!
interface Loopback0
 description "Router-ID"
 ip address <LOOPBACK_IPv4> 255.255.255.255
!
mpls ip
```

### 4.2 Transit Interface Configuration Template

Configure all point-to-point links between core devices.

```bash
interface <INTERFACE_ID>
 description "<Device> to <Linked-Device>"
 ip address <IPv4_ADDRESS> 255.255.255.254
 ospf network point-to-point
!
 mpls ip
!
 ospfv3 10 ipv4 area 0
!
 no shutdown
```

IPv6 configuration is in [`03_IPv6_DualStack.md`](/enhancements/03_IPv6_DualStack.md).

### 4.3 OSPFv3 Process Configuration

Configure OSPFv3 with dual address-family support on all core devices:

```bash
router ospfv3 10
!
 address-family ipv4 unicast
  router-id <LOOPBACK_IPv4>
  passive-interface default
  no passive-interface <P2P interfaces>
```

### 4.4 MPLS Label Distribution Protocol (LDP)

Enabling LDP to distribute labels for IGP prefixes, and configuration of Penultimate Hop Popping (PHP).

**Apply LDP configuration to these interfaces on each device:**

| Device      | LDP-Enabled Interfaces                              |
|-------------|-----------------------------------------------------|
| R1_OSLO     | G4 (RR2), G5 (RR1)                                  |
| R2_BGO      | G4 (RR2), G5 (RR1)                                  |
| RR1_OSLO    | G1 (R1), G3 (R2), G5 (RR2), G3 (CORE1), G4 (CORE2)  |
| RR2_BGO     | G3 (R1), G1 (R2), G5 (RR2), G2 (CORE1), G1 (CORE2)  |
| CORE1_OSLO  | G1 (RR1), G2 (RR2), G4 (CORE2)                      |
| CORE2_BGO   | G2 (RR1), G1 (RR2), G4 (CORE2)                      |

### Global LDP Configuration (Apply to ALL 6 core devices)

```bash
mpls ldp router-id <IPv4_LOOPBACK> force
!
interface <INTERFACE_ID>
 mpls ldp igp sync
 mpls ldp discovery transport-address <IPv4_LOOPBACK>           #L0 of device
!
mpls ldp advertise-labels
mpls ldp explicit-null
```
### 4.5 BGP Route Reflector Hierarchy

#### Route Reflector Configuration (RR1_OSLO and RR2_BGO)

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
  neighbor 10.255.1.1 peer-group RR-CLIENTS             # R1 Loopback0
  neighbor 10.255.1.5 peer-group RR-CLIENTS             # R2 Loopback0
!
  neighbor <OTHER_RR_LOOPBACK> remote-as 65001          # E.g. 10.255.1.4 (RR2 L0, if config of RR1)
  neighbor <OTHER_RR_LOOPBACK> update-source Loopback0
  neighbor <OTHER_RR_LOOPBACK> send-community both
!
```

**RR Client Assignments:**
- **RR1_OSLO (10.255.1.3)** serves clients: R1_OSLO (10.255.1.1), CORE1_OSLO (10.255.1.5)
- **RR2_BGO (10.255.1.4)** serves clients: R2_BGO (10.255.1.2), CORE2_BGO (10.255.1.6)

#### Route Reflector Client Configuration

* Appllied to R1_OSLO, R2_BGO, CORE1_OSLO, CORE2_BGO

All four client devices peer with both route reflectors (RR1_OSLO and RR2_BGO) for redundancy.

```bash
router bgp 65001
 bgp router-id <CLIENT_LOOPBACK_IPv4>
 bgp log-neighbor-changes
!
 address-family ipv4 unicast
  neighbor 10.255.1.3 remote-as 65001               # Peer with RR2   
  neighbor 10.255.1.3 update-source Loopback0
  neighbor 10.255.1.3 send-community both
!
  neighbor 10.255.1.4 remote-as 65001               # Peer with RR2
  neighbor 10.255.1.4 update-source Loopback0
  neighbor 10.255.1.4 send-community both
```

---

## 5. Verification and Validation

### 5.1 MPLS LDP and Label Distribution Validation

Verify that MPLS forwarding entries exist for all loopback prefixes:

```bash
show mpls ldp neighbor
show mpls ldp bindings
show mpls ldp discovery
show mpls forwarding-table
show mpls forwarding-table detail
```

**Expected Results:**
- LDP neighbors should show **Oper** state for all core device adjacencies
- Label bindings should exist for all /32 loopback prefixes (`10.255.1.1-10.255.1.6`)
- Forwarding entries showing **imp-null** (implicit-null) for directly connected destinations
- Pop Label behavior for penultimate hop routers in MPLS paths


### 5.2 MPLS Forwarding Table Validation

Confirm that MPLS forwarding entries exist for all loopback prefixes:

```bash
show mpls forwarding-table
show mpls forwarding-table detail
```

**Expected Results:**
- Forwarding entries for 10.255.1.1/32 through 10.255.1.6/32
- Local labels in range 16-1048575 (platform-dependent)
- Outgoing labels showing **imp-null** for directly connected neighbors
- Next-hop IP addresses matching P2P link addressing

### 5.3 iBGP Session State Verification

Validate BGP session establishment and route reflector functionality:

```bash
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors <neighbor-ip> advertised-routes
show bgp ipv4 unicast neighbors <neighbor-ip> received-routes
```

**Expected Results:**
- All BGP sessions in **Established** state
- Route reflectors should show multiple advertised prefixes to clients
- Clients should receive reflected routes from other RR clients
- No BGP sessions should be in **Idle**, **Connect**, or **Active** states

### 5.4 End-to-End Reachability Testing

Test IP connectivity between all core device loopbacks:

```bash
ping 10.255.1.1 source 10.255.1.6
ping 10.255.1.2 source 10.255.1.5  
ping 10.255.1.3 source 10.255.1.4
traceroute 10.255.1.1 source 10.255.1.6
```

**Expected Results:**
- 100% success rate for all ping tests
- Round-trip times < 10ms within lab environment
- Traceroute paths should follow optimal OSPF cost calculations
- No packet loss or timeout conditions

---

## 6. Troubleshooting Common Issues

### BGP Session Failures
- **Symptom**: BGP neighbors stuck in **Active** or **Connect** state
- **Resolution**: Verify IP reachability via `ping` and confirm `update-source Loopback0` configuration

### MPLS Forwarding Gaps
- **Symptom**: Missing LFIB entries for specific prefixes
- **Resolution**: Verify `mpls ip` is configured on transit interfaces and OSPF is advertising loopbacks

### Route Reflector Problems
- **Symptom**: RR clients not receiving reflected routes
- **Resolution**: Confirm `route-reflector-client` configuration and verify cluster-id uniqueness

---

## 7. Rollback Procedure

To return the network to pre-BGP, pre-MPLS baseline configuration:

```bash
## Remove BGP configuration
no router bgp 65001
!
## Remove LDP configuration
no mpls ldp router-id
no mpls ldp advertise-labels  
no mpls ldp explicit-null
!
## Remove OSPFv3 configuration  
no router ospfv3 10
!
## Remove MPLS and LDP from all interfaces
interface <interface-id>
 no mpls ip
 no mpls ldp igp sync
 no mpls ldp discovery transport-address
 no ospfv3 10 ipv4 area 0
 no ospfv3 10 ipv6 area 0
!
## Disable global MPLS
no mpls ip
```

Execute these commands on all six core network devices to restore pre-implementation state.