# Step 03 - eBGP Session Establishment

This step establishes external BGP sessions between AS 65001 and upstream providers, peering partners, and customer networks using **IOS-XR** and **IOSv**. The eBGP configuration provides internet connectivity, settlement-free peering relationships, and customer services while maintaining the existing iBGP route reflector hierarchy.

---

## 1. eBGP session matrix

| Local Device  | Remote Device | Relationship Type | Session Type | Local IP         | Remote IP     | Remote ASN |
|---------------|---------------|-------------------|--------------|------------------|---------------|------------|
| R1_OSLO       | ISP1_OSLO     | Customer-Provider | Transit      | 192.0.2.0/31     | 192.0.2.1     | 65002      |
| R1_OSLO       | ISP2_BGO      | Customer-Provider | Transit      | 192.0.2.2/31     | 192.0.2.3     | 65002      |
| R2_BGO        | ISP1_OSLO     | Customer-Provider | Transit      | 192.0.2.4/31     | 192.0.2.5     | 65002      |
| R2_BGO        | ISP2_BGO      | Customer-Provider | Transit      | 192.0.2.6/31     | 192.0.2.7     | 65002      |
| R1_OSLO       | CUST1_OSLO    | Provider-Customer | Customer     | 198.51.100.1/31  | 198.51.100.0  | 65003      |
| R2_BGO        | CUST2_BGO     | Provider-Customer | Customer     | 198.51.100.5/31  | 198.51.100.4  | 65003      |
| CORE1_OSLO    | PEER1_OSLO    | Peer-Peer         | Public       | 192.0.2.26/31    | 192.0.2.27    | 65010      |
| CORE2_BGO     | PEER2_BGO     | Peer-Peer         | Public       | 192.0.2.28/31    | 192.0.2.29    | 65011      |

Refer to [`ASN_Plan.md`](/docs/ASN_Plan.md) for the complete ASN assignments and business relationships.

The existing **RR1_OSLO** and **RR2_BGO** hierarchy remains unchanged. The eBGP learned routes are distributed via iBGP to all core devices with route filtering and manipulation applied at eBGP boundaries.

---

## 2. Implementation

### 2.1 Upstream Provider Sessions

These new configs may look a bit complex, but it does follow basic python-like structure, looking at it, the configuration is configuring inbound and outbound filtering policies for the customer BGP session using **prefix-sets** and **route-policies** on XR, these are similar to *prefix-list* and *route-maps* on IOS-XE. The goal is to strictly control what prefixes are accapted from, and what prefixes are advertised to the customer through theuse of if/else-statements.

The `if/else` logic in the route-policies is acting as a filter. For **R1_OSLO**, in `CUST1_IN`, the router is checking that if a recieved prefix from the customer matches the `CUST1_IN` prefix-set of `198.51.100.0/25 le 32`. If it does, then the route is accapted (`pass`). If the route is outside the prefix-set `CUST1_IN`, then it is **rejected** (`drop`). The same logic applies to `CUST1_OUT`, which only allows advertisement of the default route `0.0.0.0/0`. This way it avoids route-leaks and accidental advertisement. Same concept as IOS-XE, just written in a cleaner, more code-like format. This is mirrored on the other network devices.

**R1_OSLO Configuration:**

Configure customer policies
```bash
prefix-set CUST1_IN
  198.51.100.0/25 le 32
end-set
!
prefix-set CUST1_OUT
  0.0.0.0/0
end-set
!
route-policy CUST1_IN
  if destination in CUST1_IN then
    pass
  else
    drop
  endif
end-policy
!
route-policy CUST1_OUT
  if destination in CUST1_OUT then
    pass
  else
    drop
  endif
end-policy
```

**Configure the BGP sessions:**
```bash
router bgp 65001
 neighbor 192.0.2.1
  remote-as 65002
  description "Upstream to ISP1_OSLO"
  address-family ipv4 unicast
   send-community-ebgp
   send-extended-community-ebgp
  !
 !
 neighbor 192.0.2.3
  remote-as 65002
  description "Upstream to ISP2_BGO"
  address-family ipv4 unicast
   send-community-ebgp
   send-extended-community-ebgp
  !
 !
 neighbor 198.51.100.0
  remote-as 65003
  description "Customer CUST1_OSLO"
  address-family ipv4 unicast
   send-community-ebgp
   send-extended-community-ebgp
   default-originate
   route-policy CUST1_IN in
   route-policy CUST1_OUT out
```

**R2_BGO Configuration:**
```bash
prefix-set CUST2_IN
  198.51.100.128/25 le 32
end-set
!
prefix-set CUST2_OUT
  0.0.0.0/0
end-set
!
route-policy CUST2_IN
  if destination in CUST2_IN then
    pass
  else
    drop
  endif
end-policy
!
route-policy CUST2_OUT
  if destination in CUST2_OUT then
    pass
  else
    drop
  endif
end-policy
```

**Configure BGP sessions:**
```bash
router bgp 65001
 neighbor 192.0.2.5
  remote-as 65002
  description "Upstream to ISP1_OSLO"
  address-family ipv4 unicast
   send-community-ebgp
   send-extended-community-ebgp
  !
 !
 neighbor 192.0.2.7
  remote-as 65002
  description "Upstream to ISP2_BGO"
  address-family ipv4 unicast
   send-community-ebgp
   send-extended-community-ebgp
  !
 !
 neighbor 198.51.100.4
  remote-as 65003
  description "Customer CUST2_BGO"
  address-family ipv4 unicast
   send-community-ebgp
   send-extended-community-ebgp
   default-originate
   route-policy CUST2_IN in
   route-policy CUST2_OUT out
```

### 2.2 BGP peering sessions for CORE to PEER nodes

**CORE1_OSLO Configuration:**

```bash
router bgp 65001
 neighbor 192.0.2.27
  remote-as 65010
  description "Peer with PEER1_OSLO"
  address-family ipv4 unicast
   send-community-ebgp
   send-extended-community-ebgp
```

**CORE2_BGO Configuration:**
```bash
router bgp 65001
 neighbor 192.0.2.29
  remote-as 65011
  description "Peer with PEER2_BGO"
  address-family ipv4 unicast
   send-community-ebgp
   send-extended-community-ebgp
```

## 3. External Device Configurations

These configurations are specifically for the external devices (meaning **non-core** devices), to establish a complete BGP session. Only the six **core** devices run **IOS-XR 9000v**, while the **non-core** are using **IOSv**. The configuration will be different.

### 3.1 ISP1_OSLO Configuration (Upstream Provider)
```bash
router bgp 65002
 bgp router-id 10.255.1.7
 bgp log-neighbor-changes
 network 10.255.1.7 mask 255.255.255.255
!
 neighbor 192.0.2.0 remote-as 65001
 neighbor 192.0.2.0 description "R1_OSLO"
 neighbor 192.0.2.0 send-community both
!
 neighbor 192.0.2.4 remote-as 65001
 neighbor 192.0.2.4 description "R2_BGO"
 neighbor 192.0.2.4 send-community both
```

### 3.2 ISP2_BGO Configuration (Upstream Provider)
```bash
router bgp 65002
 bgp router-id 10.255.1.8
 bgp log-neighbor-changes
 network 10.255.1.8 mask 255.255.255.255
!
 neighbor 192.0.2.2 remote-as 65001
 neighbor 192.0.2.2 description "R1_OSLO"
 neighbor 192.0.2.2 send-community both
!
 neighbor 192.0.2.6 remote-as 65001
 neighbor 192.0.2.6 description "R2_BGO"
 neighbor 192.0.2.6 send-community both
```

### 3.3 PEER1_OSLO Configuration (Peering Partner)
```bash
router bgp 65010
 bgp router-id 10.255.1.9
 bgp log-neighbor-changes
 network 10.255.1.9 mask 255.255.255.255
 !
 neighbor 192.0.2.26 remote-as 65001
 neighbor 192.0.2.26 description "CORE1_OSLO"
 neighbor 192.0.2.26 send-community both
```

### 3.4 PEER2_BGO Configuration (Peering Partner)
```bash
router bgp 65011
 bgp router-id 10.255.1.10
 bgp log-neighbor-changes
 network 10.255.1.10 mask 255.255.255.255
 !
 neighbor 192.0.2.28 remote-as 65001
 neighbor 192.0.2.28 description "CORE2_BGO"
 neighbor 192.0.2.28 send-community both
```

### 3.5 CUST1_OSLO Configuration (Customer)
```bash
router bgp 65003
 bgp router-id 10.255.1.11
 bgp log-neighbor-changes
 network 10.255.1.11 mask 255.255.255.255
 !
 neighbor 198.51.100.1 remote-as 65001
 neighbor 198.51.100.1 description "R1_OSLO"
 neighbor 198.51.100.1 send-community both
 !
 neighbor 198.51.100.3 remote-as 65003
 neighbor 198.51.100.3 description "CUST_CORE_OSLO"
 neighbor 198.51.100.3 send-community both
```

### 3.6 CUST2_BGO Configuration (Customer)
```bash
router bgp 65003
 bgp router-id 10.255.1.12
 bgp log-neighbor-changes
 network 10.255.1.12 mask 255.255.255.255
 !
 neighbor 198.51.100.5 remote-as 65001
 neighbor 198.51.100.5 description "R2_BGO"
 neighbor 198.51.100.5 send-community both
 !
 neighbor 198.51.100.9 remote-as 65003
 neighbor 198.51.100.9 description "CUST_CORE_OSLO"
 neighbor 198.51.100.9 send-community both
```

### 3.7 CUST_CORE_OSLO Configuration (Customer)
```bash
router bgp 65003
 bgp router-id 10.255.1.13
 bgp log-neighbor-changes
 network 10.255.1.13 mask 255.255.255.255
 !
 neighbor 198.51.100.2 remote-as 65003
 neighbor 198.51.100.2 description "CUST1_OSLO"
 neighbor 198.51.100.2 send-community both
 !
 neighbor 198.51.100.8 remote-as 65003
 neighbor 198.51.100.8 description "CUST2_OSLO"
 neighbor 198.51.100.8 send-community both
```

---

## 4. Verification

### 4.1 eBGP Session State

Verify that all external BGP sessions are established:
```bash
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors <neighbor-ip> routes
```

All eBGP sessions should show an `Established` state with the uptime counter increasing without reset. The received routes should include upstream, peer, or customer prefixes (depending on which device is being checked), along with the correct next-hop information.

---

## 5. Troubleshooting

**eBGP Session Establishment Failures:** If the BGP neighbors are stuck in either the `Active` or `Connect` state, that would mean that there's an issue with either the IP connectivity, interface state, or a BGP configuration syntax error.

**Route Advertisement Issues:** If the correct routes are not advertised to eBGP neighbors, it would mean there's either than incorrect network statement, or a prefix-list misconfiguration.

**Next-Hop Reachability Issues:** If routes are installed but the traffic fails, then the IGP is not properly advertising eBGP next-hop addresses.

---

## 6. Rollback

To remove all eBGP sessions and revert to iBGP-only:

### 6.1 Non-core network devices:
```bash
## PEER1_OSLO 
no router bgp 65010
!
## PEER2_BGO 
no router bgp 65011
!
## ISP1_OSLO and ISP2_BGO 
no router bgp 65002
!
## CUST1_OSLO, CUST2_BGO, and CUST_CORE_OSLO
no router bgp 65003
```

### 6.2 CORE1_OSLO:
```bash
router bgp 65001
 address-family ipv4 unicast
  no neighbor 192.0.2.27
```

### 6.3 CORE2_BGO:
```bash
router bgp 65001
 address-family ipv4 unicast
  no neighbor 192.0.2.29
```

### 6.5 R1_OSLO:
```bash
router bgp 65001
 address-family ipv4 unicast
  no neighbor 192.0.2.1
  no neighbor 192.0.2.3
  no neighbor 198.51.100.0
 !
no prefix-set CUST1_IN
no prefix-set CUST1_OUT
no route-policy CUST1_IN
no route-policy CUST1_OUT
```

### 6.6 R2_BGO:
```bash
router bgp 65001
 no neighbor 192.0.2.5
 no neighbor 192.0.2.7
 no neighbor 198.51.100.4
 !
no prefix-set CUST2_IN
no prefix-set CUST2_OUT
no route-policy CUST2_IN
no route-policy CUST2_OUT
```
