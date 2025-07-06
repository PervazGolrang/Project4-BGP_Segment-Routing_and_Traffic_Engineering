# Step 03 - eBGP Session Establishment

This step establishes external BGP sessions between AS 65001 and upstream providers, peering partners, and customer networks. The eBGP configuration provides internet connectivity, settlement-free peering relationships, and customer services while maintaining the existing iBGP route reflector hierarchy.

---

## 1. Objective

This phase implements external BGP connectivity to complete the service provider routing infrastructure:

1. **Upstream Transit**: Establish BGP sessions with ISP1_OSLO and ISP2_BGO for internet connectivity
2. **Public Peering**: Configure settlement-free peering with PEER1_OSLO and PEER2_BGO  
3. **Customer Services**: Provide BGP services to CUST1_OSLO and CUST2_BGO
4. **Route Propagation**: Ensure proper route advertisement and filtering between eBGP and iBGP domains

---

## 2. Network Scope

### eBGP Session Matrix

| Local Device  | Remote Device | Relationship Type | Session Type | Local IP      | Remote IP     | Remote ASN |
|---------------|---------------|-------------------|--------------|---------------|---------------|------------|
| R1_OSLO       | ISP1_OSLO     | Customer-Provider | Transit      | 192.0.2.0/31  | 192.0.2.1     | 65002      |
| R1_OSLO       | ISP2_BGO      | Customer-Provider | Transit      | 192.0.2.2/31  | 192.0.2.3     | 65002      |
| R2_BGO        | ISP1_OSLO     | Customer-Provider | Transit      | 192.0.2.4/31  | 192.0.2.5     | 65002      |
| R2_BGO        | ISP2_BGO      | Customer-Provider | Transit      | 192.0.2.6/31  | 192.0.2.7     | 65002      |
| R1_OSLO       | CUST1_OSLO    | Provider-Customer | Customer     | 198.51.100.1  | 198.51.100.0  | 65003      |
| R2_BGO        | CUST2_BGO     | Provider-Customer | Customer     | 198.51.100.7  | 198.51.100.6  | 65003      |
| CORE1_OSLO    | PEER1_OSLO    | Peer-Peer         | Public       | 192.0.2.26/31 | 192.0.2.27    | 65010      |
| CORE2_BGO     | PEER2_BGO     | Peer-Peer         | Public       | 192.0.2.28/31 | 192.0.2.29    | 65011      |

Refer to xxx as a reminder.

### BGP Route Reflector Integration
- **iBGP Infrastructure**: Existing RR1_OSLO and RR2_BGO hierarchy remains unchanged
- **Route Distribution**: eBGP learned routes distributed via iBGP to all core devices
- **Policy Application**: Route filtering and manipulation applied at eBGP boundaries

---

## 3. Prerequisites

- **Step 01 Complete**: OSPFv3, LDP, and iBGP route reflector hierarchy operational
- **Step 02 Complete**: SR-MPLS enabled on all core devices with node-SID advertisement
- **Interface Configuration**: All P2P links configured per IP_Plan.md
- **IGP Reachability**: Full IP connectivity between all eBGP session endpoints

---

## 4. Implementation Procedure

### 4.1 Upstream Provider Sessions

Configuring BGP sessions to upstream ISPs for internet transit services.

#### R1_OSLO Upstream Configuration

```bash
router bgp 65001
 bgp router-id 10.255.1.1
 bgp log-neighbor-changes
!
 neighbor 192.0.2.1 description "Upstream to ISP1_OSLO"
 neighbor 192.0.2.1 remote-as 65002
 neighbor 192.0.2.1 ebgp-multihop 1
 neighbor 192.0.2.1 send-community both
!
 neighbor 192.0.2.3 description "Upstream to ISP2_BGO"
 neighbor 192.0.2.3 remote-as 65002
 neighbor 192.0.2.3 ebgp-multihop 1
 neighbor 192.0.2.3 send-community both
!
 address-family ipv4 unicast
  neighbor 192.0.2.1 activate
  neighbor 192.0.2.3 activate
  network 192.0.2.0 mask 255.255.255.0
  network 198.51.100.0 mask 255.255.255.0
```

#### R2_BGO Upstream Configuration

```bash
router bgp 65001
 bgp router-id 10.255.1.2
 bgp log-neighbor-changes
!
 neighbor 192.0.2.5 description "Upstream to ISP1_OSLO"
 neighbor 192.0.2.5 remote-as 65002
 neighbor 192.0.2.5 ebgp-multihop 1
 neighbor 192.0.2.5 send-community both
!
 neighbor 192.0.2.7 description "Upstream to ISP2_BGO"
 neighbor 192.0.2.7 remote-as 65002
 neighbor 192.0.2.7 ebgp-multihop 1
 neighbor 192.0.2.7 send-community both
 !
 address-family ipv4 unicast
  neighbor 192.0.2.5 activate
  neighbor 192.0.2.7 activate
  network 192.0.2.0 mask 255.255.255.0
  network 198.51.100.0 mask 255.255.255.0
```

### 4.2 Public Peering Sessions

Configure BGP sessions with public peering partners for direct route exchange.

#### CORE1_OSLO Peering Configuration

```bash
router bgp 65001
 bgp router-id 10.255.1.5
 bgp log-neighbor-changes
 !
 neighbor 192.0.2.27 remote-as 65010
 neighbor 192.0.2.27 description "Downlink to PEER1_OSLO"
 neighbor 192.0.2.27 ebgp-multihop 1
 neighbor 192.0.2.27 send-community both
 !
 address-family ipv4 unicast
  neighbor 192.0.2.27 activate
  network 192.0.2.0 mask 255.255.255.0
  network 198.51.100.0 mask 255.255.255.0
```

#### CORE2_BGO Peering Configuration

```bash
router bgp 65001
 bgp router-id 10.255.1.6
 bgp log-neighbor-changes
 !
 neighbor 192.0.2.29 remote-as 65011
 neighbor 192.0.2.29 description "Downlink to PEER2_BGO"
 neighbor 192.0.2.29 ebgp-multihop 1
 neighbor 192.0.2.29 send-community both
 !
 address-family ipv4 unicast
  neighbor 192.0.2.29 activate
  network 192.0.2.0 mask 255.255.255.0
  network 198.51.100.0 mask 255.255.255.0
```

### 4.3 Basic Prefix Lists

Configure basic prefix lists for customer route filtering:

#### R1_OSLO Customer Configuration

```bash
ip prefix-list CUST1_IN seq 5 permit 198.51.100.0/25 le 32
ip prefix-list CUST1_IN seq 100 deny 0.0.0.0/0 le 32
!
ip prefix-list CUST1_OUT seq 5 permit 0.0.0.0/0
ip prefix-list CUST1_OUT seq 100 deny 0.0.0.0/0 le 32
```

#### R2_BGO Customer Configuration
```bash
ip prefix-list CUST2_IN seq 5 permit 198.51.100.128/25 le 32
ip prefix-list CUST2_IN seq 100 deny 0.0.0.0/0 le 32
!
ip prefix-list CUST2_OUT seq 5 permit 0.0.0.0/0  
ip prefix-list CUST2_OUT seq 100 deny 0.0.0.0/0 le 32
```

Adding an explicit deny at `sequence 100` was made to provide flexibility for future configuration changes.

### 4.4 Customer Sessions

Configure BGP sessions to provide transit services to customer networks.

#### R1_OSLO Customer Configuration

```bash
router bgp 65001
 !
 neighbor 198.51.100.0 description "Uplink to CUST1_OSLO"
 neighbor 198.51.100.0 remote-as 65003
 neighbor 198.51.100.0 ebgp-multihop 1
 neighbor 198.51.100.0 send-community both
 !
 address-family ipv4 unicast
  neighbor 198.51.100.0 activate
  neighbor 198.51.100.0 default-originate
  neighbor 198.51.100.0 prefix-list CUST1_IN in
  neighbor 198.51.100.0 prefix-list CUST1_OUT out
```

#### R2_BGO Customer Configuration

```bash
router bgp 65001
 !
 neighbor 198.51.100.6 remote-as 65003
 neighbor 198.51.100.6 description "Uplink to CUST2_BGO"
 neighbor 198.51.100.6 ebgp-multihop 1
 neighbor 198.51.100.6 send-community both
 !
 address-family ipv4 unicast
  neighbor 198.51.100.6 activate
  neighbor 198.51.100.6 default-originate
  neighbor 198.51.100.6 prefix-list CUST2_IN in
  neighbor 198.51.100.6 prefix-list CUST2_OUT out
```

---

## 5. Verification and Validation

### 5.1 eBGP Session State Verification

Verify all external BGP sessions are established:

```bash
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors
show bgp ipv4 unicast neighbors <neighbor-ip> received-routes
```

**Expected Results:**
- All eBGP sessions show **Established** state
- **Uptime** counters incrementing without resets
- **Received routes** showing upstream/peer/customer prefixes appropriately

### 5.2 Route Advertisement Verification

Confirm proper route advertisement to each neighbor type:

```bash
show bgp ipv4 unicast neighbors <upstream-ip> advertised-routes
show bgp ipv4 unicast neighbors <peer-ip> advertised-routes  
show bgp ipv4 unicast neighbors <customer-ip> advertised-routes
```

**Expected Results:**
- **Upstream ISPs**: Receive customer and own prefixes only
- **Public Peers**: Receive customer and own prefixes only (no transit routes)
- **Customers**: Receive default route and/or full BGP table

### 5.3 iBGP Route Propagation

Verify eBGP routes propagate correctly through iBGP:

```bash
show bgp ipv4 unicast
show bgp ipv4 unicast regexp 65002
show bgp ipv4 unicast regexp 65003
show bgp ipv4 unicast regexp 65010
```

**Expected Results:**
- **Upstream routes** (AS 65002) visible on all core devices via iBGP
- **Customer routes** (AS 65003) advertised with proper AS-path
- **Peer routes** (AS 65010, 65011) distributed through route reflectors

### 5.4 End-to-End Connectivity Testing

Test connectivity through different path types:

```bash
# Test via upstream transit
ping 8.8.8.8 source 10.255.1.1
traceroute 8.8.8.8 source 10.255.1.1
!
# Test customer connectivity  
ping 198.51.100.10 source 10.255.1.1
traceroute 198.51.100.10 source 10.255.1.1
```

**Expected Results:**
- **Internet connectivity** via upstream providers successful
- **Customer prefixes** reachable via direct eBGP sessions
- **Traceroute paths** follow optimal BGP path selection

### 5.5 BGP Best Path Selection

Verify BGP best path selection follows expected criteria:

```bash
show bgp ipv4 unicast <prefix>
show bgp ipv4 unicast <prefix> bestpath
show bgp ipv4 unicast rib-failure
```

**Expected Results:**
- **Best path selection** follows local-preference, AS-path length, and IGP metric
- **No RIB failures** due to routing loops or invalid next-hops
- **Multiple paths** available for redundancy

---

## 6. Troubleshooting Common Issues

### eBGP Session Establishment Failures
- **Symptom**: BGP neighbors stuck in **Active** or **Connect** state
- **Resolution**: Verify IP connectivity, interface states, and BGP configuration syntax

### Route Advertisement Issues
- **Symptom**: Expected routes not advertised to eBGP neighbors
- **Resolution**: Check network statements, route-maps, and prefix-list configurations

### iBGP Route Propagation Problems
- **Symptom**: eBGP learned routes not visible on other iBGP speakers
- **Resolution**: Verify route reflector configuration and next-hop reachability

### AS-Path Loop Detection
- **Symptom**: Routes rejected due to own ASN in AS-path
- **Resolution**: Check for configuration errors and unintended route reflection

### Next-Hop Reachability Issues
- **Symptom**: Routes installed but traffic fails
- **Resolution**: Verify IGP advertisement of eBGP next-hop addresses

---

## 7. Rollback Procedure

To remove all eBGP sessions and revert to iBGP-only operation:

```bash
# Remove customer sessions
router bgp 65001
 address-family ipv4 unicast
  no neighbor 198.51.100.0
  no neighbor 198.51.100.6
!
# Remove peering sessions  
router bgp 65001
 address-family ipv4 unicast
  no neighbor 192.0.2.27
  no neighbor 192.0.2.29
!
# Remove upstream sessions
router bgp 65001
 address-family ipv4 unicast
  no neighbor 192.0.2.1
  no neighbor 192.0.2.3
  no neighbor 192.0.2.5
  no neighbor 192.0.2.7
!
# Remove prefix lists
no ip prefix-list CUST1_IN
no ip prefix-list CUST1_OUT
no ip prefix-list CUST2_IN
no ip prefix-list CUST2_OUT
```

Execute these commands on the appropriate devices to restore iBGP-only operation.