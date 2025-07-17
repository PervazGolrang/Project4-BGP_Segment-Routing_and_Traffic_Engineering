# Step 03 - eBGP Session Establishment

This step establishes external BGP sessions between AS 65001 and upstream providers, peering partners, and customer networks. The eBGP configuration provides internet connectivity, settlement-free peering relationships, and customer services while maintaining the existing iBGP route reflector hierarchy.

---

## 1. Objective

This phase implements external BGP connectivity to complete the service provider routing infrastructure:

1. **Upstream Transit:** Establish BGP sessions with ISP1_OSLO and ISP2_BGO for internet connectivity
2. **Public Peering:** Configure settlement-free peering with PEER1_OSLO and PEER2_BGO
3. **Customer Services:** Provide BGP services to CUST1_OSLO and CUST2_BGO
4. **Route Propagation:** Ensure proper route advertisement and filtering between eBGP and iBGP domains

---

## 2. eBGP session matrix

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

Refer to [`ASN_Plan.md`](/docs/ASN_Plan.md) for the complete ASN assignments and business relationships.

The existing **RR1_OSLO** and **RR2_BGO** hierarchy remains unchanged. The eBGP learned routes are distributed via iBGP to all core devices with route filtering and manipulation applied at eBGP boundaries.

---

## 3. Implementation

### 3.1 Upstream Provider Sessions

Configure BGP sessions to upstream ISPs for internet transit services.

**R1_OSLO upstream configuration (connects to upstreams via G2 and G3):**

```bash
router bgp 65001
 bgp router-id 10.255.1.1
 bgp log-neighbor-changes
!
 neighbor 192.0.2.1 description "Upstream to ISP1_OSLO"
 neighbor 192.0.2.1 remote-as 65002
 neighbor 192.0.2.1 send-community both
!
 neighbor 192.0.2.3 description "Upstream to ISP2_BGO"
 neighbor 192.0.2.3 remote-as 65002
 neighbor 192.0.2.3 send-community both
!
 address-family ipv4 unicast
  neighbor 192.0.2.1 activate
  neighbor 192.0.2.3 activate
  network 192.0.2.0 mask 255.255.255.0
  network 198.51.100.0 mask 255.255.255.0
```

**R2_BGO upstream configuration (connects to upstreams via G2 and G3):**

```bash
router bgp 65001
 bgp router-id 10.255.1.2
 bgp log-neighbor-changes
!
 neighbor 192.0.2.5 description "Upstream to ISP1_OSLO"
 neighbor 192.0.2.5 remote-as 65002
 neighbor 192.0.2.5 send-community both
!
 neighbor 192.0.2.7 description "Upstream to ISP2_BGO"
 neighbor 192.0.2.7 remote-as 65002
 neighbor 192.0.2.7 send-community both
!
 address-family ipv4 unicast
  neighbor 192.0.2.5 activate
  neighbor 192.0.2.7 activate
  network 192.0.2.0 mask 255.255.255.0
  network 198.51.100.0 mask 255.255.255.0
```

### 3.2 Public Peering Sessions

Configure BGP sessions with public peering partners for direct route exchange.

**CORE1_OSLO peering configuration (connects to PEER1_OSLO via G3):**

```bash
router bgp 65001
 bgp router-id 10.255.1.5
 bgp log-neighbor-changes
!
 neighbor 192.0.2.27 remote-as 65010
 neighbor 192.0.2.27 description "Peer with PEER1_OSLO"
 neighbor 192.0.2.27 send-community both
!
 address-family ipv4 unicast
  neighbor 192.0.2.27 activate
  network 192.0.2.0 mask 255.255.255.0
  network 198.51.100.0 mask 255.255.255.0
```

**CORE2_BGO peering configuration (connects to PEER2_BGO via G3):**

```bash
router bgp 65001
 bgp router-id 10.255.1.6
 bgp log-neighbor-changes
!
 neighbor 192.0.2.29 remote-as 65011
 neighbor 192.0.2.29 description "Peer with PEER2_BGO"
 neighbor 192.0.2.29 send-community both
!
 address-family ipv4 unicast
  neighbor 192.0.2.29 activate
  network 192.0.2.0 mask 255.255.255.0
  network 198.51.100.0 mask 255.255.255.0
```

### 3.3 Basic Prefix Lists

Configure basic prefix lists for customer route filtering.

**R1_OSLO customer configuration:**

```bash
ip prefix-list CUST1_IN seq 5 permit 198.51.100.0/25 le 32
ip prefix-list CUST1_IN seq 100 deny 0.0.0.0/0 le 32
!
ip prefix-list CUST1_OUT seq 5 permit 0.0.0.0/0
ip prefix-list CUST1_OUT seq 100 deny 0.0.0.0/0 le 32
```

**R2_BGO customer configuration:**

```bash
ip prefix-list CUST2_IN seq 5 permit 198.51.100.128/25 le 32
ip prefix-list CUST2_IN seq 100 deny 0.0.0.0/0 le 32
!
ip prefix-list CUST2_OUT seq 5 permit 0.0.0.0/0  
ip prefix-list CUST2_OUT seq 100 deny 0.0.0.0/0 le 32
```

Having an explicit deny at sequence 100 is best-practice for future configuration, rather than a simple sequence 10, causing annoyance if further prefixes would be added.

### 3.4 Customer Sessions

Configure BGP sessions to provide transit services to customer networks.

**R1_OSLO customer configuration (connects to CUST1_OSLO via G1):**

```bash
router bgp 65001
!
 neighbor 198.51.100.0 description "Customer CUST1_OSLO"
 neighbor 198.51.100.0 remote-as 65003
 neighbor 198.51.100.0 send-community both
!
 address-family ipv4 unicast
  neighbor 198.51.100.0 activate
  neighbor 198.51.100.0 default-originate
  neighbor 198.51.100.0 prefix-list CUST1_IN in
  neighbor 198.51.100.0 prefix-list CUST1_OUT out
```

**R2_BGO customer configuration (connects to CUST2_BGO via G1):**

```bash
router bgp 65001
!
 neighbor 198.51.100.6 remote-as 65003
 neighbor 198.51.100.6 description "Customer CUST2_BGO"
 neighbor 198.51.100.6 send-community both
!
 address-family ipv4 unicast
  neighbor 198.51.100.6 activate
  neighbor 198.51.100.6 default-originate
  neighbor 198.51.100.6 prefix-list CUST2_IN in
  neighbor 198.51.100.6 prefix-list CUST2_OUT out
```

---

## 4. Verification

### 4.1 eBGP Session State

Verify that all the external BGP sessions are established:

```bash
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors
show bgp ipv4 unicast neighbors <neighbor-ip> received-routes
```

All eBGP sessions should show an `Established` state with an uptime counter incrementing without reset. The received routes should show upstream/peer/costumer prefixes.

### 4.2 Route Advertisement

Confirm correct route advertisement to each neighbor type:

```bash
show bgp ipv4 unicast neighbors <upstream-ip> advertised-routes
show bgp ipv4 unicast neighbors <peer-ip> advertised-routes  
show bgp ipv4 unicast neighbors <customer-ip> advertised-routes
```

Upstream ISPs should receive customer and own prefixes only. Public peers should receive customer and own prefixes only (no transit routes). Customers should receive either a default route and/or a full BGP table.


### 4.3 iBGP Route Propagation

Verify that the eBGP routes correctly propagate through iBGP:

```bash
show bgp ipv4 unicast
show bgp ipv4 unicast regexp 65002
show bgp ipv4 unicast regexp 65003
show bgp ipv4 unicast regexp 65010
```

Upstream routes (AS 65002) should be visible on all core devices via iBGP. Customer routes (AS 65003) should be advertised with proper AS-path, and peer routes (AS 65010, 65011) should be distributed through route reflectors.

### 4.4 End-to-End Connectivity Testing

Test the connectivity through the different path types:

```bash
ping 198.51.100.10 source 10.255.1.1
traceroute 198.51.100.10 source 10.255.1.1
```

Customer prefixes should be reachable via the direct eBGP sessions. The traceroute paths should follow the optimal BGP path selection.

### 4.5 BGP Best Path Selection

Verify that the BGP best-path-selection follows the expected criteria:

```bash
show bgp ipv4 unicast <prefix>
show bgp ipv4 unicast <prefix> bestpath
show bgp ipv4 unicast rib-failure
```

The best-path-selection should follow the local-preference, AS-path length, and IGP metric. There shouldn't be any RIB failures due to either the routing loops or an invalid next-hop, with multiple paths available for redundancy.

---

## 5. Troubleshooting

**eBGP Session Establishment Failures:** BGP neighbors that are stuck in `Active` or `Connect` state would mean issues with either IP connectivity issues, interface states, or a BGP configuration syntax error.

**Route Advertisement Issues:** If the correct routes are not advertised to eBGP neighbors, it would result from incorrect network statements, or prefix-list configurations.

**iBGP Route Propagation Problems:** eBGP learned routes that are not visible on the other iBGP speakers would mean that the route reflector configuration or next-hop reachability is incorrectly configured.

**Next-Hop Reachability Issues:** If routes are installed but the traffic fails, it would indicate that the IGP is not properly advertising eBGP next-hop addresses.

---

## 6. Rollback

To remove all eBGP sessions and revert to iBGP-only:

```bash
router bgp 65001
 address-family ipv4 unicast
  no neighbor 198.51.100.0
  no neighbor 198.51.100.6
  no neighbor 192.0.2.27
  no neighbor 192.0.2.29
  no neighbor 192.0.2.1
  no neighbor 192.0.2.3
  no neighbor 192.0.2.5
  no neighbor 192.0.2.7
!
no ip prefix-list CUST1_IN
no ip prefix-list CUST1_OUT
no ip prefix-list CUST2_IN
no ip prefix-list CUST2_OUT
```