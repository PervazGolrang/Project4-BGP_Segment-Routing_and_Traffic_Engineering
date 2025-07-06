# IP Address Plan

This IP-PÅlan follows a clear, structured model using /31 or /30 for point-to-point links, /32 for loopbacks, and documentation prefixes.

## Global Address Blocks

* **IPv4 Core Block**: `192.0.2.0/24`
* **IPv4 Customer Block**: `198.51.100.0/24`
* **IPv6 Core Block**: `2001:db8:face::/48`
* **IPv6 Customer Block**: `2001:db8:cust::/48`

---

## Loopback Addresses

| Device          | IPv4 Loopback  | IPv6 Loopback      | SID Index |
| --------------- | -------------- | ------------------ | --------- |
| R1_OSLO         | 10.255.1.1     | 2001:db8:face::1   | 16001     |
| R2_BGO          | 10.255.1.2     | 2001:db8:face::2   | 16002     |
| RR1_OSLO        | 10.255.1.3     | 2001:db8:face::3   | 16011     |
| RR2_BGO         | 10.255.1.4     | 2001:db8:face::4   | 16012     |
| CORE1_OSLO      | 10.255.1.5     | 2001:db8:face::5   | 16021     |
| CORE2_BGO       | 10.255.1.6     | 2001:db8:face::6   | 16022     |
| ISP1_OSLO       | 10.255.1.7     | 2001:db8:face::7   | —         |
| ISP2_BGO        | 10.255.1.8     | 2001:db8:face::8   | —         |
| PEER1_OSLO      | 10.255.1.9     | 2001:db8:face::9   | —         |
| PEER2_BGO       | 10.255.1.10    | 2001:db8:face::10  | —         |
| CUST1_OSLO      | 10.255.1.11    | 2001:db8:cust::11  | —         |
| CUST2_BGO       | 10.255.1.12    | 2001:db8:cust::12  | —         |
| CUST_CORE_OSLO  | 10.255.1.13    | 2001:db8:cust::13  | —         |
| CUST_SRV1_OSLO  | 10.255.1.14    | 2001:db8:cust::14  | —         |

IPv6 Loopback block can be ignored if not following [`03_IPv6_DualStack.md`](/enhancements/03_IPv6_DualStack.md), it is only mentioned here as a visual representation.

---

## Point-to-Point Links

| Link                              | Subnet           | Interface A     | Interface B     |
| --------------------------------- | ---------------- | --------------- | --------------- |
| R1_OSLO ↔ ISP1_OSLO               | 192.0.2.0/31     | .0 (G2)         | .1 (G2)         |
| R1_OSLO ↔ ISP2_BGO                | 192.0.2.2/31     | .2 (G3)         | .3 (G3)         |
| R2_BGO ↔ ISP1_OSLO                | 192.0.2.4/31     | .4 (G3)         | .5 (G3)         |
| R2_BGO ↔ ISP2_BGO                 | 192.0.2.6/31     | .6 (G2)         | .7 (G2)         |
| R1_OSLO ↔ RR1_OSLO                | 192.0.2.8/31     | .8 (G4)         | .9 (G1)         |
| R1_OSLO ↔ RR2_BGO                 | 192.0.2.10/31    | .10 (G5)        | .11 (G3)        |
| R2_BGO ↔ RR1_OSLO                 | 192.0.2.12/31    | .12 (G5)        | .13 (G3)        |
| R2_BGO ↔ RR2_BGO                  | 192.0.2.14/31    | .14 (G4)        | .15 (G1)        |
| RR1_OSLO ↔ CORE1_OSLO             | 192.0.2.16/31    | .16 (G2)        | .17 (G1)        |
| RR1_OSLO ↔ CORE2_BGO              | 192.0.2.18/31    | .18 (G4)        | .19 (G2)        |
| RR2_BGO ↔ CORE1_OSLO              | 192.0.2.20/31    | .20 (G4)        | .21 (G2)        |
| RR2_BGO ↔ CORE2_BGO               | 192.0.2.22/31    | .22 (G2)        | .23 (G1)        |
| RR1_OSLO ↔ RR2_BGO                | 198.0.2.24/31    | .24 (G5)        | .25 (G5)        |
| CORE1_OSLO ↔ PEER1_OSLO           | 192.0.2.26/31    | .26 (G3)        | .27 (G1)        |
| CORE2_BGO ↔ PEER2_BGO             | 192.0.2.28/31    | .28 (G3)        | .29 (G1)        |
| CUST1_OSLO ↔ R1_OSLO              | 198.51.100.0/31  | .0 (G3)         | .1 (G1)         |
| CUST1_OSLO ↔ CUST_CORE_OSLO       | 198.51.100.2/31  | .2 (G1)         | .3 (G2)         |
| CUST_CORE_OSLO ↔ CUST_SRV1        | 198.51.100.4/31  | .4 (G1)         | .5 (E0)         |
| CUST2_BGO ↔ R2_BGO                | 198.51.100.6/31  | .6 (G3)         | .7 (G1)         |
| CORE1_OSLO ↔ CORE2_BGO            | 198.51.100.8/31  | .8 (G4)         | .9 (G4)         |

---

## Notes

* All loopbacks are /32 (IPv4) and /128 (IPv6)
* All P2P links use /31 where supported
* Global SRGB range is 16000–23999; node SID = base + router ID
* IPv6 can be dual-stack enabled later via [`03_IPv6_DualStack.md`](/enhancements/03_IPv6_DualStack.md)
