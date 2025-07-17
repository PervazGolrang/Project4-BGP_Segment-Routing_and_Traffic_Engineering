# Step 04 - BGP Policies and Route Control

This step implements comprehensive BGP route policies to enforce business relationships, optimize traffic engineering, and provide granular control over route advertisement and selection. The policies establish proper filtering, community-based route marking, and local-preference manipulation to ensure predictable routing behavior across all peering relationships.

---

## 1. Objective

This phase implements advanced BGP policy controls on the eBGP sessions:

1. **Business Relationship Enforcement:** Implement proper route filtering based on customer-provider and peer-peer relationships
2. **Community-Based Route Control:** Deploy BGP communities for route marking and policy application
3. **Traffic Engineering:** Configure local-preference and MED manipulation for optimal path selection
4. **Security Hardening:** Implement comprehensive prefix filtering and route validation
5. **Operational Flexibility:** Enable granular route control for traffic engineering and troubleshooting

---

## 2. Network policy matrix

| Device        | Neighbor Type | Policy Focus                     | Communities Used    | Local-Pref Range |
|---------------|---------------|----------------------------------|---------------------|------------------|
| R1_OSLO       | Upstream ISPs | Outbound filtering, backup path  | 65001:100, 65001:90 | 100-150          |
| R2_BGO        | Upstream ISPs | Outbound filtering, primary path | 65001:100, 65001:90 | 100-150          |
| CORE1_OSLO    | Public Peers  | Strict filtering, no transit     | 65001:200           | 200              |
| CORE2_BGO     | Public Peers  | Strict filtering, no transit     | 65001:200           | 200              |
| R1_OSLO       | Customers     | Full table export, validation    | 65001:300           | 300              |
| R2_BGO        | Customers     | Full table export, validation    | 65001:300           | 300              |

BGO (Bergen) is used as the primary path due to it is closer to my home city, Stavanger.

**BGP community allocation:**

| Community    | Purpose                    | Applied To              | Policy Action               |
|--------------|----------------------------|-------------------------|-----------------------------|
| 65001:100    | Customer routes            | Customer-originated     | Advertise to all neighbors  |
| 65001:200    | Peer routes                | Peer-learned routes     | Advertise to only customers |
| 65001:300    | Upstream routes            | Transit-learned routes  | Advertise to only customers |
| 65001:666    | Blackhole/discard          | Blackholed prefixes     | Trigger blackhole routing   |
| 65001:90     | Backup path                | Secondary transit       | Lower local-preference      |
| 65001:110    | Primary path               | Primary transit         | Higher local-preference     |

---

## 3. Implementation

### 3.1 Global Community Lists and Route-Maps

Configure standard community lists for policy matching:

```bash
ip community-list standard CUSTOMER_ROUTES permit 65001:100
ip community-list standard PEER_ROUTES permit 65001:200
ip community-list standard UPSTREAM_ROUTES permit 65001:300
ip community-list standard BLACKHOLE_ROUTES permit 65001:666
ip community-list standard PRIMARY_PATH permit 65001:110
ip community-list standard BACKUP_PATH permit 65001:90
!
ip community-list expanded NO_EXPORT_TO_PEERS permit _65001:200_
ip community-list expanded CUSTOMER_ONLY permit _65001:[13]00_        #100 and 300
```

### 3.2 Upstream Provider Policies

**R1_OSLO upstream policy configuration:**

```bash
route-map TO_UPSTREAM_OUT permit 10
 description "Allow customer routes to upstream"
 match community CUSTOMER_ROUTES
 set community 65001:100 additive
!
route-map TO_UPSTREAM_OUT permit 20
 description "Allow own infrastructure routes"
 match ip address prefix-list OWN_PREFIXES
 set community 65001:100 additive
!
route-map TO_UPSTREAM_OUT deny 30
 description "Deny all other routes (no transit)"
!
route-map FROM_ISP1_IN permit 10
 description "Primary upstream path via ISP1"
 set community 65001:300 additive
 set local-preference 120
!
route-map FROM_ISP2_IN permit 10
 description "Backup upstream path via ISP2"
 set community 65001:300 65001:90 additive
 set local-preference 110
!
router bgp 65001
 address-family ipv4 unicast
  neighbor 192.0.2.1 route-map FROM_ISP1_IN in
  neighbor 192.0.2.1 route-map TO_UPSTREAM_OUT out
  neighbor 192.0.2.3 route-map FROM_ISP2_IN in
  neighbor 192.0.2.3 route-map TO_UPSTREAM_OUT out
```

**R2_BGO upstream policy configuration:** 

```bash
route-map FROM_ISP1_IN permit 10
 description "Backup upstream path via ISP1"
 set community 65001:300 65001:90 additive
 set local-preference 110
!
route-map FROM_ISP2_IN permit 10
 description "Primary upstream path via ISP2"
 set community 65001:300 additive  
 set local-preference 120
!
router bgp 65001
 address-family ipv4 unicast
  neighbor 192.0.2.5 route-map FROM_ISP1_IN in
  neighbor 192.0.2.5 route-map TO_UPSTREAM_OUT out
  neighbor 192.0.2.7 route-map FROM_ISP2_IN in
  neighbor 192.0.2.7 route-map TO_UPSTREAM_OUT out
```

### 3.3 Public Peering Policies

**CORE1_OSLO peering policy configuration:**

Do note these are strict outbound peering policies to peers, no transit routes.

```bash
route-map TO_PEER_OUT permit 10
 description "Allow customer routes to peers"
 match community CUSTOMER_ROUTES
 set community 65001:200 additive
!
route-map TO_PEER_OUT permit 20
 description "Allow own routes to peers"
 match ip address prefix-list OWN_PREFIXES
 set community 65001:200 additive
!
route-map TO_PEER_OUT deny 30
 description "Deny upstream and peer routes"
!
route-map FROM_PEER_IN permit 10
 description "Accept peer routes with validation"
 match ip address prefix-list PEER_PREFIXES
 set community 65001:200 additive
 set local-preference 200
!
route-map FROM_PEER_IN deny 20
 description "Deny invalid prefixes from peers"
!
router bgp 65001
 address-family ipv4 unicast
  neighbor 192.0.2.27 route-map FROM_PEER_IN in
  neighbor 192.0.2.27 route-map TO_PEER_OUT out
  neighbor 192.0.2.27 maximum-prefix 1000 restart 5
```

**CORE2_BGO peering policy configuration:**

```bash
!
route-map TO_PEER_OUT permit 10
 description "Allow customer routes to peers"
 match community CUSTOMER_ROUTES
 set community 65001:200 additive
!
route-map TO_PEER_OUT permit 20
 description "Allow own routes to peers"
 match ip address prefix-list OWN_PREFIXES
 set community 65001:200 additive
!
route-map TO_PEER_OUT deny 30
 description "Deny upstream and peer routes"
!
route-map FROM_PEER_IN permit 10
 description "Accept peer routes with validation"
 match ip address prefix-list PEER_PREFIXES
 set community 65001:200 additive
 set local-preference 200
!
route-map FROM_PEER_IN deny 20
 description "Deny invalid prefixes from peers"
!
router bgp 65001
 address-family ipv4 unicast
  neighbor 192.0.2.29 route-map FROM_PEER_IN in
  neighbor 192.0.2.29 route-map TO_PEER_OUT out  
  neighbor 192.0.2.29 maximum-prefix 1000 restart 5
```

Maximum-prefix is set to 1000 to protect memory overload and CPU-usage, with an automatic session retry every 5 minutes.

### 3.4 Customer Policies

**R1_OSLO customer policy configuration:**

```bash
route-map TO_CUSTOMER_OUT permit 10
 description "Provide all routes to customer with community tagging"
 set community 65001:300 additive
!
route-map FROM_CUST1_IN permit 10
 description "Accept only customer-owned prefixes"
 match ip address prefix-list CUST1_PREFIXES
 set community 65001:100 additive
 set local-preference 300
 set origin igp
!
route-map FROM_CUST1_IN deny 20
 description "Deny all other prefixes from customer"
!
ip prefix-list CUST1_PREFIXES seq 5 permit 198.51.100.128/25 le 28
ip prefix-list CUST1_PREFIXES seq 10 deny 0.0.0.0/0 le 32
!
router bgp 65001
 address-family ipv4 unicast
  neighbor 198.51.100.0 route-map FROM_CUST1_IN in
  neighbor 198.51.100.0 route-map TO_CUSTOMER_OUT out
  neighbor 198.51.100.0 maximum-prefix 50 restart 5
```

**R2_BGO customer policy configuration:**

```bash
route-map TO_CUSTOMER_OUT permit 10
 description "Provide all routes to customer with community tagging"
 set community 65001:300 additive
!
route-map FROM_CUST2_IN permit 10
 description "Accept only customer-owned prefixes"
 match ip address prefix-list CUST2_PREFIXES
 set community 65001:100 additive
 set local-preference 300
 set origin igp
!
route-map FROM_CUST2_IN deny 20
 description "Deny all other prefixes from customer"
!
ip prefix-list CUST2_PREFIXES seq 5 permit 198.51.100.128/25 le 28
ip prefix-list CUST2_PREFIXES seq 10 deny 0.0.0.0/0 le 32
!
router bgp 65001
 address-family ipv4 unicast
  neighbor 198.51.100.6 route-map FROM_CUST2_IN in
  neighbor 198.51.100.6 route-map TO_CUSTOMER_OUT out
  neighbor 198.51.100.6 maximum-prefix 50 restart 5
```

### 3.5 Infrastructure and Security Policies

Defining prefixes and bogon filtering:

```bash
ip prefix-list OWN_PREFIXES seq 5 permit 192.0.2.0/24 le 30
ip prefix-list OWN_PREFIXES seq 10 permit 10.255.1.0/24 le 32
!
ip prefix-list PEER_PREFIXES seq 5 permit 203.0.113.0/24 le 30
ip prefix-list PEER_PREFIXES seq 10 permit 198.51.100.0/24 le 30
ip prefix-list PEER_PREFIXES seq 100 deny 0.0.0.0/0 le 32
!
ip prefix-list BOGON_PREFIXES seq 5 deny 0.0.0.0/8 le 32
ip prefix-list BOGON_PREFIXES seq 10 deny 10.0.0.0/8 le 32
ip prefix-list BOGON_PREFIXES seq 15 deny 127.0.0.0/8 le 32
ip prefix-list BOGON_PREFIXES seq 20 deny 169.254.0.0/16 le 32
ip prefix-list BOGON_PREFIXES seq 25 deny 172.16.0.0/12 le 32
ip prefix-list BOGON_PREFIXES seq 30 deny 192.0.2.0/24 le 32
ip prefix-list BOGON_PREFIXES seq 35 deny 192.168.0.0/16 le 32
ip prefix-list BOGON_PREFIXES seq 40 deny 224.0.0.0/4 le 32
ip prefix-list BOGON_PREFIXES seq 45 deny 240.0.0.0/4 le 32
ip prefix-list BOGON_PREFIXES seq 100 permit 0.0.0.0/0 le 32
```

Blackhole route handling for security incidents:

```bash
route-map BLACKHOLE_TRIGGER permit 10
 description "Trigger blackhole routing for security"
 match community BLACKHOLE_ROUTES
 set community no-export additive
 set local-preference 50
 set ip next-hop discard
!
router bgp 65001
 address-family ipv4 unicast
  table-map BLACKHOLE_TRIGGER
```

---

## 4. Verification

### 4.1 Policy Application

Verify that the route-maps are correctly applied:

```bash
show route-map
show route-map TO_UPSTREAM_OUT
show route-map FROM_ISP1_IN detail
show bgp ipv4 unicast neighbors <neighbor-ip> policy
```

Route-maps should be active on all configured neighbors with match/set statistics incrementing for active policies. Policy configuration should match the intended business relationships.

### 4.2 Community Assignment

Confirm the proper community tagging on routes:

```bash
show bgp ipv4 unicast community 65001:100
show bgp ipv4 unicast community 65001:200
show bgp ipv4 unicast community 65001:300
show bgp ipv4 unicast regexp _65001:
```

Customer routes should be tagged with 65001:100, peer routes with 65001:200, upstream routes with 65001:300. Community propagation through iBGP should be maintained.

### 4.3 Local-Preference and Path Selection

Verify the traffic engineering through local-preference manipulation:

```bash
show bgp ipv4 unicast <prefix> bestpath
show bgp ipv4 unicast <prefix>
show bgp ipv4 unicast rib-failure
```

Primary paths (local-pref 120) should be preferred over backup paths (local-pref 110). Customer routes (local-pref 300) should be preferred over peer/upstream routes. Path diversity should be maintained for redundancy.

### 4.4 Route Advertisement Filtering

Confirm the proper outbound filtering to each neighbor type:

```bash
show bgp ipv4 unicast neighbors <upstream-ip> advertised-routes
show bgp ipv4 unicast neighbors <peer-ip> advertised-routes
show bgp ipv4 unicast neighbors <customer-ip> advertised-routes
```

Upstream neighbors should receive customer + own prefixes only. Peer neighbors should receive customer + own prefixes only (no transit). Customer neighbors should receive full table or default route.

### 4.5 Security and Validation Testing

Test prefix filtering and bogon protection:

```bash
show bgp ipv4 unicast regexp _10_
show bgp ipv4 unicast regexp _127_
```

---

## 5. Troubleshooting

**Route-Map Not Applied:** If a policy is not taking effect on routes it would mean that there's a route-map syntax error, missing neighbor activation, or it needs a BGP soft-reset.

**Community Assignment Failures:** Routes that are missing expected communities would result from incorrect route-map permit/deny logic, or an incorrect community-list definition.

**Local-Preference Not Working:** Suboptimal path selection despite local-preference settings would result to iBGP propagation issues or higher-priority BGP attributes overriding.

**Prefix Filtering Issues:** Unexpected routes accepted or denied result from incorrect prefix-list definitions or route-map sequence numbers.

---

## 6. Rollback

To remove all BGP policies and revert to basic session configuration:

```bash
router bgp 65001
 address-family ipv4 unicast
  no neighbor <neighbor-ip> route-map <route-map-name> in
  no neighbor <neighbor-ip> route-map <route-map-name> out
  no neighbor <neighbor-ip> maximum-prefix
!
no route-map TO_UPSTREAM_OUT
no route-map FROM_ISP1_IN
no route-map FROM_ISP2_IN
no route-map TO_PEER_OUT
no route-map FROM_PEER_IN
no route-map TO_CUSTOMER_OUT
no route-map FROM_CUST1_IN
no route-map FROM_CUST2_IN
!
no ip community-list standard CUSTOMER_ROUTES
no ip community-list standard PEER_ROUTES
no ip community-list standard UPSTREAM_ROUTES
no ip prefix-list OWN_PREFIXES
no ip prefix-list BOGON_PREFIXES
no ip prefix-list CUST1_PREFIXES
no ip prefix-list CUST2_PREFIXES
```