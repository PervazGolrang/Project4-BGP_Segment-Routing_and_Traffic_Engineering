# Step 04 - BGP Policies and Route Control

This step implements comprehensive BGP route policies using **IOS-XR** to enforce business relationships, optimize traffic engineering, and provide granular control over route advertisement and selection. The policies establish proper filtering, community-based route marking, and local-preference manipulation to ensure predictable routing behavior across all peering relationships.

---

## 1. Network policy matrix

| Device        | Neighbor Type | Policy Focus                     | Communities Used    | Local-Pref Range |
|---------------|---------------|----------------------------------|---------------------|------------------|
| R1_OSLO       | Upstream ISPs | Outbound filtering, backup path  | 65001:100, 65001:90 | 100-150          |
| R2_BGO        | Upstream ISPs | Outbound filtering, primary path | 65001:100, 65001:90 | 100-150          |
| CORE1_OSLO    | Public Peers  | Strict filtering, no transit     | 65001:200           | 200              |
| CORE2_BGO     | Public Peers  | Strict filtering, no transit     | 65001:200           | 200              |
| R1_OSLO       | Customers     | Full table export, validation    | 65001:300           | 300              |
| R2_BGO        | Customers     | Full table export, validation    | 65001:300           | 300              |

**BGP community allocation:**

| Community    | Purpose                    | Applied To              | Policy Action               |
|--------------|----------------------------|-------------------------|-----------------------------|
| 65001:100    | Customer routes            | Customer-originated     | Advertise to all neighbors  |
| 65001:200    | Peer routes                | Peer-learned routes     | Advertise to only customers |
| 65001:300    | Upstream routes            | Transit-learned routes  | Advertise to only customers |
| 65001:90     | Backup path                | Secondary transit       | Lower local-preference      |
| 65001:110    | Primary path               | Primary transit         | Higher local-preference     |

---

## 2. Implementation

This step follows the concepts from Step03, as it sets up prefix-sets, but now also community-sets to filter and tag routes between the upstream providers and the customer. `OWN_PREFIXES` is just what is locally owned, and `CUST1_PREFIXES` is what the customer is allowed to send. There are two community-sets:
- One for routes learned from customers: `65001:100`
- One for routes learned from the upstream ISPs: `65001:300`

The route-policies use basic logic. When receiving routes from ISP1 and ISP2, they get tagged with `UPSTREAM_ROUTES` and are given a local-pref where ISP1 is prefered, (ISP1 = 120), and (ISP2 = 110). For the customer, it checks if the prefix is in `CUST1_PREFIXES`. If it is, then it's tagged as customer, it gets a local-pref of 300 (so it's preferred over the upstreams), origin is set to IGP, and it is **passed**. If it's not in the prefix-set, it is **dropped**.

For outbound routes going to the ISPs, prefixes are only allowed if they came from customers or from `OWN_PREFIXES`, else they are **droppped**. These are also tagged with the customer community. When sending routes to the customer, it doesn't filter, but it tags them as upstream and passes them. This setup avoids leaks, adds simple community tagging for tracking, and gives preference to customer-learned prefixes over upstream ones. The same logic is mirrored to the other network devices, with slightly different configurations. They all follow the **Network policy matrix** and **BGP community allocation** at the top of the document.

### 2.1 R1_OSLO Configuration (Upstream + Customer)

**Configuring the prefix- and community sets:**

```bash
prefix-set OWN_PREFIXES
  192.0.2.0/24 le 30,
  10.255.1.0/24 le 32
end-set
!
prefix-set CUST1_PREFIXES
  198.51.100.0/25 le 32
end-set
!
community-set CUSTOMER_ROUTES
  65001:100
end-set
!
community-set UPSTREAM_ROUTES
  65001:300
end-set
```

**Configuring route policies:**

```bash
route-policy TO_UPSTREAM_OUT
  if community matches-any CUSTOMER_ROUTES then
    set community (65001:100) additive
    pass
  elseif destination in OWN_PREFIXES then
    set community (65001:100) additive
    pass
  else
    drop
  endif
end-policy
!
route-policy FROM_ISP1_IN
  set community (65001:300) additive
  set local-preference 120
  pass
end-policy
!
route-policy FROM_ISP2_IN
  set community (65001:300, 65001:90) additive
  set local-preference 110
  pass
end-policy
!
route-policy FROM_CUST1_IN
  if destination in CUST1_PREFIXES then
    set community (65001:100) additive
    set local-preference 300
    set origin igp
    pass
  else
    drop
  endif
end-policy
!
route-policy TO_CUSTOMER_OUT
  set community (65001:300) additive
  pass
end-policy
```

**Apply policies to BGP neighbors:**

```bash
router bgp 65001
 neighbor 192.0.2.1
  address-family ipv4 unicast
   route-policy FROM_ISP1_IN in
   route-policy TO_UPSTREAM_OUT out
  !
 !
 neighbor 192.0.2.3
  address-family ipv4 unicast
   route-policy FROM_ISP2_IN in
   route-policy TO_UPSTREAM_OUT out
  !
 !
 neighbor 198.51.100.0
  address-family ipv4 unicast
   route-policy FROM_CUST1_IN in
   route-policy TO_CUSTOMER_OUT out
   maximum-prefix 50 75 restart 5
```

Each neighbor gets the correct inbound and outbound policy. For the customer neighbor a `maximum-prefix 50 75 restart 5` is added. It means that the router will allow up to `50` prefixes from the customer. If it recieves more than `75%` of that (~38 prefixes), it gives a warning. If it hits `50`, the session is shutdown and auto-restarts after 5 minutes. It is a safety limit to avoid route floods.

### 2.2 R2_BGO Configuration (Upstream + Customer)

**Configure prefix sets and community sets:**

```bash
prefix-set OWN_PREFIXES
  192.0.2.0/24 le 30,
  10.255.1.0/24 le 32
end-set
!
prefix-set CUST2_PREFIXES
  198.51.100.128/25 le 32
end-set
!
community-set CUSTOMER_ROUTES
  65001:100
end-set
!
community-set UPSTREAM_ROUTES
  65001:300
end-set
```

**Configure route policies:**

```bash
route-policy TO_UPSTREAM_OUT
  if community matches-any CUSTOMER_ROUTES then
    set community (65001:100) additive
    pass
  elseif destination in OWN_PREFIXES then
    set community (65001:100) additive
    pass
  else
    drop
  endif
end-policy
!
route-policy FROM_ISP1_IN
  set community (65001:300, 65001:90) additive
  set local-preference 110
  pass
end-policy
!
route-policy FROM_ISP2_IN
  set community (65001:300) additive
  set local-preference 120
  pass
end-policy
!
route-policy FROM_CUST2_IN
  if destination in CUST2_PREFIXES then
    set community (65001:100) additive
    set local-preference 300
    set origin igp
    pass
  else
    drop
  endif
end-policy
!
route-policy TO_CUSTOMER_OUT
  set community (65001:300) additive
  pass
end-policy
```

**Apply policies to BGP neighbors:**

```bash
router bgp 65001
 neighbor 192.0.2.5
  address-family ipv4 unicast
   route-policy FROM_ISP1_IN in
   route-policy TO_UPSTREAM_OUT out
  !
 !
 neighbor 192.0.2.7
  address-family ipv4 unicast
   route-policy FROM_ISP2_IN in
   route-policy TO_UPSTREAM_OUT out
  !
 !
 neighbor 198.51.100.4
  address-family ipv4 unicast
   route-policy FROM_CUST2_IN in
   route-policy TO_CUSTOMER_OUT out
   maximum-prefix 50 75 restart 5
```

### 2.3 CORE1_OSLO Configuration (Peering Only)

**Configure prefix sets and community sets:**

```bash
prefix-set OWN_PREFIXES
  192.0.2.0/24 le 30,
  10.255.1.0/24 le 32
end-set
!
prefix-set PEER_PREFIXES
  203.0.113.0/24 le 30,
  198.51.100.0/24 le 30
end-set
!
prefix-set BOGON_PREFIXES
  0.0.0.0/8 le 32,
  10.0.0.0/8 le 32,
  127.0.0.0/8 le 32,
  169.254.0.0/16 le 32,
  172.16.0.0/12 le 32,
  192.168.0.0/16 le 32,
  224.0.0.0/4 le 32,
  240.0.0.0/4 le 32
end-set
!
community-set CUSTOMER_ROUTES
  65001:100
end-set
!
community-set PEER_ROUTES
  65001:200
end-set
```

**Configure strict peering policies:**

```bash
route-policy TO_PEER_OUT
  if community matches-any CUSTOMER_ROUTES then
    set community (65001:200) additive
    pass
  elseif destination in OWN_PREFIXES then
    set community (65001:200) additive
    pass
  else
    drop
  endif
end-policy
!
route-policy FROM_PEER_IN
  if destination in PEER_PREFIXES and not destination in BOGON_PREFIXES then
    set community (65001:200) additive
    set local-preference 200
    pass
  else
    drop
  endif
end-policy
```

**Apply policies to peering neighbor:**

```bash
router bgp 65001
 neighbor 192.0.2.27
  address-family ipv4 unicast
   route-policy FROM_PEER_IN in
   route-policy TO_PEER_OUT out
   maximum-prefix 1000 75 restart 5
```

### 2.4 CORE2_BGO Configuration (Peering Only)

**Configure prefix sets and community sets (same as CORE1_OSLO):**

```bash
prefix-set OWN_PREFIXES
  192.0.2.0/24 le 30,
  10.255.1.0/24 le 32
end-set
!
prefix-set PEER_PREFIXES
  203.0.113.0/24 le 30,
  198.51.100.0/24 le 30
end-set
!
prefix-set BOGON_PREFIXES
  0.0.0.0/8 le 32,
  10.0.0.0/8 le 32,
  127.0.0.0/8 le 32,
  169.254.0.0/16 le 32,
  172.16.0.0/12 le 32,
  192.168.0.0/16 le 32,
  224.0.0.0/4 le 32,
  240.0.0.0/4 le 32
end-set
!
community-set CUSTOMER_ROUTES
  65001:100
end-set
!
community-set PEER_ROUTES
  65001:200
end-set
```

**Configure and apply same peering policies:**

```bash
route-policy TO_PEER_OUT
  if community matches-any CUSTOMER_ROUTES then
    set community (65001:200) additive
    pass
  elseif destination in OWN_PREFIXES then
    set community (65001:200) additive
    pass
  else
    drop
  endif
end-policy
!
route-policy FROM_PEER_IN
  if destination in PEER_PREFIXES and not destination in BOGON_PREFIXES then
    set community (65001:200) additive
    set local-preference 200
    pass
  else
    drop
  endif
end-policy
```

**Apply policies to peering neighbor:**

```bash
router bgp 65001
 neighbor 192.0.2.29
  address-family ipv4 unicast
   route-policy FROM_PEER_IN in
   route-policy TO_PEER_OUT out
   maximum-prefix 1000 75 restart 5
```

---

## 3. Verification

### 3.1 Policy Application Status

Verify that the route-policies are actually applied to the correct neighbors:
```bash
show rpl
show rpl route-policy TO_UPSTREAM_OUT
show bgp neighbors 192.0.2.1 policy
show bgp policy route-policy TO_UPSTREAM_OUT
```

### 3.2 Community Assignment Verification

Check that the correct communities are being tagged on received and advertised routes:
```bash
show bgp community 65001:100
show bgp community 65001:200
show bgp community 65001:300
show bgp regexp _65001:
```

These do depend on the network device where the mentioned community-set was configured on. Customer routes should be tagged with `65001:100`, peer routes with `65001:200`, and upstream routes with `65001:300`.

### 3.3 Route Advertisement Filtering

Confirm that outbound filtering is working and routes are being sent only to the right neighbors:

```bash
show bgp neighbors <upstream-ip> advertised-routes
show bgp neighbors <peer-ip> advertised-routes
show bgp neighbors <customer-ip> advertised-routes
```

---

## 4. Troubleshooting

**Policy Not Applied**: Check route-policy syntax, BGP neighbor configuration, and soft-reset requirements.

**Community Assignment Issues**: Verify community-set definitions and route-policy logic with `show bgp policy statistics`.

**Local-Preference Problems**: Check iBGP propagation and higher-priority BGP attributes.

**Prefix Filtering Failures**: Validate prefix-set definitions and route-policy conditional logic.

---

## 5. Rollback

### 5.1 R1_OSLO:
```bash
no prefix-set OWN_PREFIXES
no prefix-set CUST1_PREFIXES
no community-set CUSTOMER_ROUTES
no community-set UPSTREAM_ROUTES
!
no route-policy TO_UPSTREAM_OUT
no route-policy FROM_ISP1_IN
no route-policy FROM_ISP2_IN
no route-policy FROM_CUST1_IN
no route-policy TO_CUSTOMER_OUT
!
router bgp 65001
 neighbor 192.0.2.1
  address-family ipv4 unicast
   no route-policy FROM_ISP1_IN in
   no route-policy TO_UPSTREAM_OUT out
 !
 neighbor 192.0.2.3
  address-family ipv4 unicast
   no route-policy FROM_ISP2_IN in
   no route-policy TO_UPSTREAM_OUT out
 !
 neighbor 198.51.100.0
  address-family ipv4 unicast
   no route-policy FROM_CUST1_IN in
   no route-policy TO_CUSTOMER_OUT out
   no maximum-prefix 50 75 restart 5
```

### 5.2 R2_BGO:
```bash
no prefix-set OWN_PREFIXES
no prefix-set CUST2_PREFIXES
no community-set CUSTOMER_ROUTES
no community-set UPSTREAM_ROUTES
!
no route-policy TO_UPSTREAM_OUT
no route-policy FROM_ISP1_IN
no route-policy FROM_ISP2_IN
no route-policy FROM_CUST1_IN
no route-policy TO_CUSTOMER_OUT
!
router bgp 65001
 neighbor 192.0.2.5
  address-family ipv4 unicast
   route-policy FROM_ISP1_IN in
   route-policy TO_UPSTREAM_OUT out
 !
 neighbor 192.0.2.7
  address-family ipv4 unicast
   route-policy FROM_ISP2_IN in
   route-policy TO_UPSTREAM_OUT out
 !
 neighbor 198.51.100.4
  address-family ipv4 unicast
   route-policy FROM_CUST2_IN in
   route-policy TO_CUSTOMER_OUT out
   maximum-prefix 50 75 restart 5
```

### 5.3 CORE1_OSLO:
```bash
no prefix-set OWN_PREFIXES
no prefix-set PEER_PREFIXES
no prefix-set BOGON_PREFIXES
no community-set CUSTOMER_ROUTES
no community-set UPSTREAM_ROUTES
!
no route-policy TO_PEER_OUT
no route-policy FROM_PEER_IN
!
router bgp 65001
 neighbor 192.0.2.27
  address-family ipv4 unicast
   route-policy FROM_PEER_IN in
   route-policy TO_PEER_OUT out
   maximum-prefix 1000 75 restart 5
```

### 5.4 CORE2_BGO:
```bash
no prefix-set OWN_PREFIXES
no prefix-set PEER_PREFIXES
no prefix-set BOGON_PREFIXES
no community-set CUSTOMER_ROUTES
no community-set UPSTREAM_ROUTES
!
no route-policy TO_PEER_OUT
no route-policy FROM_PEER_IN
!
router bgp 65001
 neighbor 192.0.2.29
  address-family ipv4 unicast
   route-policy FROM_PEER_IN in
   route-policy TO_PEER_OUT out
   maximum-prefix 1000 75 restart 5
```