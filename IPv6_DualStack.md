# IPv6 Dual-Stack Enhancement

This enhancement adds IPv6 support to the existing IPv4-only BGP Segment Routing lab, creating a dual-stack environment where both IPv4 and IPv6 operate simultaneously using the same infrastructure.

---

## 1. IPv6 Address Plan

### 1.1 IPv6 Loopback Addresses

| Device          | IPv6 Loopback         | IPv4 Loopback  |
| --------------- | --------------------- | -------------- |
| R1_OSLO         | 2001:db8:face::1/128  | 10.255.1.1/32  |
| R2_BGO          | 2001:db8:face::2/128  | 10.255.1.2/32  |
| RR1_OSLO        | 2001:db8:face::3/128  | 10.255.1.3/32  |
| RR2_BGO         | 2001:db8:face::4/128  | 10.255.1.4/32  |
| CORE1_OSLO      | 2001:db8:face::5/128  | 10.255.1.5/32  |
| CORE2_BGO       | 2001:db8:face::6/128  | 10.255.1.6/32  |
| ISP1_OSLO       | 2001:db8:face::7/128  | 10.255.1.7/32  |
| ISP2_BGO        | 2001:db8:face::8/128  | 10.255.1.8/32  |
| PEER1_OSLO      | 2001:db8:face::9/128  | 10.255.1.9/32  |
| PEER2_BGO       | 2001:db8:face::10/128 | 10.255.1.10/32 |
| CUST1_OSLO      | 2001:db8:cu00::11/128 | 10.255.1.11/32 |
| CUST2_BGO       | 2001:db8:cu00::12/128 | 10.255.1.12/32 |

### 1.2 IPv6 Point-to-Point Links

| Link                              | IPv6 Subnet                | Interface A              | Interface B              |
| --------------------------------- | -------------------------- | ------------------------ | ------------------------ |
| R1_OSLO ↔ ISP1_OSLO               | 2001:db8:face:1::/127      | ::0/127                  | ::1/127                  |
| R1_OSLO ↔ ISP2_BGO                | 2001:db8:face:2::/127      | ::0/127                  | ::1/127                  |
| R2_BGO ↔ ISP1_OSLO                | 2001:db8:face:3::/127      | ::0/127                  | ::1/127                  |
| R2_BGO ↔ ISP2_BGO                 | 2001:db8:face:4::/127      | ::0/127                  | ::1/127                  |
| R1_OSLO ↔ RR1_OSLO                | 2001:db8:face:10::/127     | ::0/127                  | ::1/127                  |
| R1_OSLO ↔ RR2_BGO                 | 2001:db8:face:11::/127     | ::0/127                  | ::1/127                  |
| R2_BGO ↔ RR1_OSLO                 | 2001:db8:face:12::/127     | ::0/127                  | ::1/127                  |
| R2_BGO ↔ RR2_BGO                  | 2001:db8:face:13::/127     | ::0/127                  | ::1/127                  |
| RR1_OSLO ↔ CORE1_OSLO             | 2001:db8:face:20::/127     | ::0/127                  | ::1/127                  |
| RR1_OSLO ↔ CORE2_BGO              | 2001:db8:face:21::/127     | ::0/127                  | ::1/127                  |
| RR2_BGO ↔ CORE1_OSLO              | 2001:db8:face:22::/127     | ::0/127                  | ::1/127                  |
| RR2_BGO ↔ CORE2_BGO               | 2001:db8:face:23::/127     | ::0/127                  | ::1/127                  |
| RR1_OSLO ↔ RR2_BGO                | 2001:db8:face:30::/127     | ::0/127                  | ::1/127                  |
| CORE1_OSLO ↔ PEER1_OSLO           | 2001:db8:face:40::/127     | ::0/127                  | ::1/127                  |
| CORE2_BGO ↔ PEER2_BGO             | 2001:db8:face:41::/127     | ::0/127                  | ::1/127                  |
| CUST1_OSLO ↔ R1_OSLO              | 2001:db8:cu00:1::/127      | ::0/127                  | ::1/127                  |
| CUST2_BGO ↔ R2_BGO                | 2001:db8:cu00:2::/127      | ::0/127                  | ::1/127                  |
| CORE1_OSLO ↔ CORE2_BGO            | 2001:db8:face:50::/127     | ::0/127                  | ::1/127                  |

---

## 2. Implementation

### 2.1 IPv6 Loopback Configuration

Configure IPv6 loopbacks on all devices:

```bash
interface Loopback0
 ipv6 enable
 ipv6 address <IPv6_LOOPBACK>/128
 ospfv3 10 ipv6 area 0
```

### 2.2 IPv6 Interface Configuration

Configure IPv6 on all point-to-point links:

```bash
interface <INTERFACE_ID>
 ipv6 enable
 ipv6 address <IPv6_ADDRESS>/127
 ospfv3 10 ipv6 area 0
```

### 2.3 OSPFv3 IPv6 Configuration

Enable IPv6 address family in OSPFv3:

```bash
router ospfv3 10
 address-family ipv6 unicast
  router-id <IPv4_ROUTER_ID>
  passive-interface default
  no passive-interface <P2P interfaces>
 exit-address-family
```

### 2.4 IPv6 BGP Configuration

#### Route Reflector IPv6 Configuration

```bash
router bgp 65001
 address-family ipv6 unicast
  neighbor RR-CLIENTS-V6 peer-group
  neighbor RR-CLIENTS-V6 remote-as 65001
  neighbor RR-CLIENTS-V6 update-source Loopback0
  neighbor RR-CLIENTS-V6 route-reflector-client
  neighbor RR-CLIENTS-V6 send-community both
!
  neighbor 2001:db8:face::1 peer-group RR-CLIENTS-V6
  neighbor 2001:db8:face::2 peer-group RR-CLIENTS-V6
  neighbor 2001:db8:face::5 peer-group RR-CLIENTS-V6
  neighbor 2001:db8:face::6 peer-group RR-CLIENTS-V6
!
  neighbor <OTHER_RR_IPv6> remote-as 65001
  neighbor <OTHER_RR_IPv6> update-source Loopback0
  neighbor <OTHER_RR_IPv6> send-community both
```

#### Client IPv6 Configuration

```bash
router bgp 65001
 address-family ipv6 unicast
  neighbor 2001:db8:face::3 remote-as 65001
  neighbor 2001:db8:face::3 update-source Loopback0
  neighbor 2001:db8:face::3 send-community both
!
  neighbor 2001:db8:face::4 remote-as 65001
  neighbor 2001:db8:face::4 update-source Loopback0
  neighbor 2001:db8:face::4 send-community both
 exit-address-family
```

### 2.5 IPv6 eBGP Sessions

#### Upstream Provider IPv6 Sessions

```bash
router bgp 65001
 neighbor 2001:db8:face:1::1 remote-as 65002
 neighbor 2001:db8:face:1::1 description "IPv6 Upstream to ISP1_OSLO"
 neighbor 2001:db8:face:1::1 send-community both
!
 address-family ipv6 unicast
  neighbor 2001:db8:face:1::1 activate
  network 2001:db8:face::/48
  network 2001:db8:cu00::/48
```

#### Customer IPv6 Sessions

```bash
router bgp 65001
 neighbor 2001:db8:cu00:1::0 remote-as 65003
 neighbor 2001:db8:cu00:1::0 description "IPv6 Customer CUST1_OSLO"
 neighbor 2001:db8:cu00:1::0 send-community both
!
 address-family ipv6 unicast
  neighbor 2001:db8:cu00:1::0 activate
  neighbor 2001:db8:cu00:1::0 default-originate
  neighbor 2001:db8:cu00:1::0 prefix-list CUST1_IPv6_IN in
  neighbor 2001:db8:cu00:1::0 prefix-list CUST1_IPv6_OUT out
```

#### Peering IPv6 Sessions

```bash
router bgp 65001
 neighbor 2001:db8:face:40::1 remote-as 65010
 neighbor 2001:db8:face:40::1 description "IPv6 Peer with PEER1_OSLO"
 neighbor 2001:db8:face:40::1 send-community both
!
 address-family ipv6 unicast
  neighbor 2001:db8:face:40::1 activate
  network 2001:db8:face::/48
  network 2001:db8:cu00::/48
```

### 2.6 IPv6 Prefix Lists and Policies

#### Customer IPv6 Prefix Lists

```bash
ipv6 prefix-list CUST1_IPv6_IN seq 5 permit 2001:db8:cu00:100::/56 le 64
ipv6 prefix-list CUST1_IPv6_IN seq 100 deny ::/0 le 128
!
ipv6 prefix-list CUST1_IPv6_OUT seq 5 permit ::/0
ipv6 prefix-list CUST1_IPv6_OUT seq 100 deny ::/0 le 128
```

#### IPv6 Route-Maps

```bash
route-map FROM_ISP1_IPv6_IN permit 10
 description "Primary IPv6 upstream path via ISP1"
 set community 65001:300 additive
 set local-preference 120
!
route-map TO_UPSTREAM_IPv6_OUT permit 10
 description "Allow customer IPv6 routes to upstream"
 match community CUSTOMER_ROUTES
 set community 65001:100 additive
!
route-map TO_UPSTREAM_IPv6_OUT permit 20
 description "Allow own IPv6 infrastructure routes"
 match ipv6 address prefix-list OWN_IPv6_PREFIXES
 set community 65001:100 additive
!
route-map TO_UPSTREAM_IPv6_OUT deny 30
 description "Deny all other IPv6 routes"
```

#### IPv6 Infrastructure Prefix Lists

```bash
ipv6 prefix-list OWN_IPv6_PREFIXES seq 5 permit 2001:db8:face::/48 le 64
ipv6 prefix-list OWN_IPv6_PREFIXES seq 10 permit 2001:db8:cust::/48 le 64
!
ipv6 prefix-list PEER_IPv6_PREFIXES seq 5 permit 2001:db8:peer1::/48 le 64
ipv6 prefix-list PEER_IPv6_PREFIXES seq 10 permit 2001:db8:peer2::/48 le 64
ipv6 prefix-list PEER_IPv6_PREFIXES seq 100 deny ::/0 le 128
```

---

## 3. Verification

### 3.1 IPv6 Connectivity Testing

Test IPv6 reachability between all core devices:

```bash
ping ipv6 2001:db8:face::1 source 2001:db8:face::6
ping ipv6 2001:db8:face::2 source 2001:db8:face::5
traceroute ipv6 2001:db8:face::1 source 2001:db8:face::6
```

### 3.2 IPv6 BGP Verification

Verify IPv6 BGP sessions and route advertisement:

```bash
show bgp ipv6 unicast summary
show bgp ipv6 unicast neighbors
show bgp ipv6 unicast neighbors <ipv6-neighbor> advertised-routes
show bgp ipv6 unicast neighbors <ipv6-neighbor> received-routes
```

### 3.3 IPv6 Route Propagation

Confirm IPv6 routes propagate through iBGP:

```bash
show bgp ipv6 unicast
show bgp ipv6 unicast regexp 65002
show bgp ipv6 unicast regexp 65003
show bgp ipv6 unicast community 65001:100
```

### 3.4 OSPFv3 IPv6 Verification

Check OSPFv3 IPv6 adjacencies and database:

```bash
show ospfv3 neighbor
show ospfv3 database
show ipv6 route ospf
```

### 3.5 IPv6 Traffic Steering

Test IPv6 traffic through SR-TE policies:

```bash
ping ipv6 2001:db8:face::2 source 2001:db8:face::1 size 1400
traceroute ipv6 2001:db8:face::2 source 2001:db8:face::1
```