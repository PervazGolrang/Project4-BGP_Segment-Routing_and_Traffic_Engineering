# Step 01 - IGP Underlay and BGP Overlay

This step establishes the foundational control-plane infrastructure for the BGP Segment-Routing Traffic Engineering lab using **IOS-XR**. All core network devices participate in a unified OSPFv3 domain to provide IP reachability, while iBGP overlay sessions are established through route reflectors to enable scalable BGP convergence without full-mesh complexity.

---

**Username is `cisco` and the password is `ciscocisco`.**

## 1. Core Network Devices

The core network consists of six devices:

| Device      | Role                | IPv4 Loopback | PID |
|-------------|---------------------|---------------|-----|
| R1_OSLO     | Provider Edge       | 10.255.1.1/32 | 10  |
| R2_BGO      | Provider Edge       | 10.255.1.2/32 | 10  |
| RR1_OSLO    | Route Reflector     | 10.255.1.3/32 | 10  |
| RR2_BGO     | Route Reflector     | 10.255.1.4/32 | 10  |
| CORE1_OSLO  | Core Transit        | 10.255.1.5/32 | 10  |
| CORE2_BGO   | Core Transit        | 10.255.1.6/32 | 10  |

---

## 2. Implementation

### 2.1 Loopback Interface Configuration

Configure to each core network device.

```bash
interface Loopback0
 description "Router-ID"
 ipv4 address <LOOPBACK_IPv4>/32
 ipv6 enable
 no shutdown
```

### 2.2 Point-to-Point Interface Configuration

Configure all P2P interfaces with /31 subnets:

```bash
interface GigabitEthernet0/0/0/1
 ipv4 address <P2P_IP>/31
 ipv6 enable
 no shutdown
```

### 2.3 OSPF Configuration

**Area 0 design** with all interfaces in backbone area:

```bash
router ospf 10
 router-id <ROUTER_ID>
 area 0
  interface Loopback0
   passive enable
  !
  interface <CORE_INTERFACES>
    network point-to-point
```

### 2.4 MPLS Configuration

Enable MPLS on all core interfaces for future SR use:

```bash
mpls ldp
 router-id <ROUTER_ID>
 interface <CORE_INTERFACES>
```

### 2.5 BGP Route Reflector Configuration

**Route reflector configuration (RR1_OSLO, RR2_BGO):**

```bash
router bgp 65001
 bgp router-id <RR_LOOPBACK_IP>
 bgp log neighbor changes detail
 address-family ipv4 unicast
  network <RR_LOOPBACK_IP>/32
 !
 neighbor-group RR-CLIENTS
  remote-as 65001
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
 !
 neighbor 10.255.1.1
  use neighbor-group RR-CLIENTS
 neighbor 10.255.1.2
  use neighbor-group RR-CLIENTS
 neighbor 10.255.1.5
  use neighbor-group RR-CLIENTS
 neighbor 10.255.1.6
  use neighbor-group RR-CLIENTS
 !
 neighbor <OTHER_RR_IP>
  remote-as 65001
  update-source Loopback0
```

### 2.6 BGP Client Configuration

**Route reflector client configuration (R1_OSLO, R2_BGO, CORE1_OSLO, CORE2_BGO):**

```bash
router bgp 65001
 bgp router-id <CLIENT_LOOPBACK_IP>
 bgp log neighbor changes detail
 address-family ipv4 unicast
  network <CLIENT_LOOPBACK_IP>/32
 !
 neighbor 10.255.1.3
  remote-as 65001
  update-source Loopback0
  address-family ipv4 unicast
  !
 neighbor 10.255.1.4
  remote-as 65001
  update-source Loopback0
  address-family ipv4 unicast
```

On IOS XR the `neighbor x.x.x.x activate` is not needed, as it automatically activates once the `address-family ipv4 unicast` and `neighbor` submods are configured.

---

## 3. Verification

### 3.1 OSPF Verification

Verify OSPF adjacencies and database:

```bash
show ospf neighbor
show ospf database
show route ipv4 ospf
```

All neighbor states should show `FULL`. The OSPF database should contain LSAs from all 6 core devices with /32 routes for all loopbacks. All core device loopbacks should be reachable via OSPF.

### 3.2 BGP Verification

Verify iBGP sessions and route reflection:

```bash
show bgp summary
show bgp ipv4 unicast
show bgp neighbour summary
```

Both Route Reflectors should show **4 clients each**, and clients learn routes from **both route reflectors**, as well as the BGP unicast neighbor, which is the redundant Route Reflector.

### 3.3 Connectivity Testing

Test end-to-end reachability:

```bash
ping 10.255.1.1 source 10.255.1.6             #Pinging R1_OSLO from CORE2_BGO Lo0
ping 10.255.1.2 source 10.255.1.5             #Pinging R2_BGO from CORE1_OSLO Lo0
traceroute 10.255.1.3 source 10.255.1.6       #Traceroute RR1_OSLO from CORE2_BGO
```

Expect a +100% success rate (after ARP), with a sub-10ms ICMP response time, and sub-20 traceroute response time in the lab environment. Ping will be lower due to routing and ARP, which is cached. While traceroute will send a new packate, through a new process in the control plane.

Expect a +100% success rate (after ARP), with a sub-10ms ICMP response time, and a completed sub-50ms traceroute hop responses in the lab environment. Ping will be faster because it uses cached routing and ARP entries. Traceroute will be slower even if `minttl 1 maxttl 1` was used, because the routes generate ICMP Time Exceeded messages in the control plane, wihch causes significant delta delays, even if it is one hop away.

- Pinging [`R1_OSLO from CORE2_BGO's Loopback0`](/wireshark/Step01-CORE2_BGO-to-R1_OSLO-ICMP.pcap) shows an average ~2ms latency between packets. Traceroute paths should follow optimal OSPF cost calculations.

- Traceroute [``RR1_OSLO from CORE2_BGO's Loopback0`](/wireshark/Step01-RR1_OSLO-to-CORE2_BGO-TRACEROUTE.pcap) shows an average ~14ms latency between packets. Traceroute followed the optimal OSPF path through cost calculations. 

### 3.4 Route Reflection Verification

Verify route reflector operation:

```bash
show bgp ipv4 unicast neighbors <client-ip> advertised-routes
show bgp ipv4 unicast neighbors <client-ip> routes
```

Clients should receive reflected routes from other clients.

---

## 4. Troubleshooting

**OSPF adjacency issues:** Check interface IP addressing and area configuration. IOS-XR requires explicit interface-to-area assignment.

**BGP session failures:** Verify loopback reachability via OSPF. Check `update-source Loopback0` configuration.

**Route reflection problems:** Confirm route-reflector-client configuration on RRs and proper neighbor-group usage.

**MPLS issues:** Verify LDP is enabled on all core interfaces and router-id is configured.

---

## 5. Rollback

To return the network to pre-BGP, pre-MPLS baseline configuration:

```bash
no router bgp 65001
no mpls ldp
no router ospf 10
```