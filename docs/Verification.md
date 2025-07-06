# Verification and Testing Plan

This document describes the testing and verification procedures for the Segment Routing (SR-MPLS) and Traffic Engineering (SR-TE) lab. It ensures that the SR infrastructure, control plane convergence, and SR-TE path behavior function as expected.

---

## 1. Node SID Reachability

Verify that each Node‑SID (refer to [`IP_plan.md`](/docs/IP_Plan.md)) is in the label table and can be pinged via MPLS.

```bash
show mpls forwarding-table | include 160
!
ping mpls ipv4 16001 repeat 3   # R1_OSLO
ping mpls ipv4 16002 repeat 3   # R2_BGO
ping mpls ipv4 16011 repeat 3   # RR1_OSLO
ping mpls ipv4 16012 repeat 3   # RR2_BGO
ping mpls ipv4 16021 repeat 3   # CORE1_OSLO
ping mpls ipv4 16022 repeat 3   # CORE2_BGO
```

---

## 2. OSPF Adjacency & Loopback Reachability

Verify OSPF (refer to [`IP_plan.md`](/docs/IP_Plan.md)) is up, and every Loopback0 (/32) is reachable.

```bash
show ip ospf neighbor
show ip ospf interface brief
!
ping 10.255.1.1 repeat 3    # R1_OSLO
ping 10.255.1.2 repeat 3    # R2_BGO
ping 10.255.1.3 repeat 3    # RR1_OSLO
ping 10.255.1.4 repeat 3    # RR2_BGO
ping 10.255.1.5 repeat 3    # CORE1_OSLO
ping 10.255.1.6 repeat 3    # CORE2_BGO
```

Verify IPv6 (if [`enhancements/03_IPv6_DualStack.md`](/enhancements/03_IPv6_DualStack.md) is deployed):

```bash
show ospfv3 neighbor
!
ping ipv6 2001:db8:face::1    # R1_OSLO
ping ipv6 2001:db8:face::2    # R2_BGO
ping ipv6 2001:db8:face::3    # RR1_OSLO
ping ipv6 2001:db8:face::4    # RR2_BGO
ping ipv6 2001:db8:face::5    # CORE1_OSLO
ping ipv6 2001:db8:face::6    # CORE2_BGO
```

---

## 3. SR-TE Policy Installation

Ensure each SR-TE policies is installed and active.

```bash
show segment-routing traffic-eng policy all
show segment-routing traffic-eng policy name TE_CORE_LOWLAT detail
```

Check:
* `Operational State : up`
* SID-list should match [`Segment-routing.md`](/docs/Segment-routing.md)
* Candidate-path preference should match [`traffic-engineering.md`](/docs/traffic-engineering.md)

---

## 4. Binding SID Resolution

SR-TE policies should install Binding-SIDs (label > SRGB):

```bash
show segment-routing traffic-eng policy binding-sid-map
show mpls forwarding-table | include BSID
```

---

## 5. MPLS Traceroute

Validate by using MPLS-aware traceroute to verify path steering.

Running SR-TE traceroute from `CORE1_OSLO` to `CORE2_BGO`

```bash
traceroute mpls ipv4 10.255.1.6 source 10.255.1.5
```

The output should display the SID stack that matches the TE policy’s SID‑list.

---

## 6. BFD

BFD should be `Up` for every protected link.

```bash
show bfd neighbors brief
```

---

## 7. Streaming Telemetry (Experiments)

Verify export of SR-TE counters:

* Ensure gNMI/NETCONF agents are enabled
* Verify gNMI subscription on port `57400`
* Check Grafana/InfluxDB dashboards for SR‑TE counters (packets, bytes, drop events).
