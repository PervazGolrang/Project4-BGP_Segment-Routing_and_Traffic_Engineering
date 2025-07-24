# IP Address Plan

This IP-Plan follows a clear, structured model using /31 or /30 for point-to-point links, /32 for loopbacks, and documentation prefixes.

## Global Address Blocks

* **IPv4 Core Block**: `192.0.2.0/24`
* **IPv4 Customer Block**: `198.51.100.0/24`

---

## Loopback Addresses

| Device          | IPv4 Loopback | SID Index | SRGB Base | SID   |
| --------------- | ------------- | --------- | --------- | ----- |
| R1_OSLO         | 10.255.1.1    | 1         | 16000     | 16001 |
| R2_BGO          | 10.255.1.2    | 2         | 16000     | 16002 |
| RR1_OSLO        | 10.255.1.3    | 11        | 16000     | 16011 |
| RR2_BGO         | 10.255.1.4    | 12        | 16000     | 16012 |
| CORE1_OSLO      | 10.255.1.5    | 21        | 16000     | 16021 |
| CORE2_BGO       | 10.255.1.6    | 22        | 16000     | 16022 |
| ISP1_OSLO       | 10.255.1.7    | —         | —         | —     |
| ISP2_BGO        | 10.255.1.8    | —         | —         | —     |
| PEER1_OSLO      | 10.255.1.9    | —         | —         | —     |
| PEER2_BGO       | 10.255.1.10   | —         | —         | —     |
| CUST1_OSLO      | 10.255.1.11   | —         | —         | —     |
| CUST2_BGO       | 10.255.1.12   | —         | —         | —     |
| CUST_CORE_OSLO  | 10.255.1.13   | —         | —         | —     |

---

## Point-to-Point Links

| Link                              | Subnet           | Interface A     | Interface B     |
| --------------------------------- | ---------------- | --------------- | --------------- |
| R1_OSLO ↔ ISP1_OSLO               | 192.0.2.0/31     | .0 (G0/2)       | .1 (G0/2)       |
| R1_OSLO ↔ ISP2_BGO                | 192.0.2.2/31     | .2 (G0/3)       | .3 (G0/3)       |
| R2_BGO ↔ ISP1_OSLO                | 192.0.2.4/31     | .4 (G0/3)       | .5 (G0/3)       |
| R2_BGO ↔ ISP2_BGO                 | 192.0.2.6/31     | .6 (G0/2)       | .7 (G0/2)       |
| R1_OSLO ↔ RR1_OSLO                | 192.0.2.8/31     | .8 (G0/4)       | .9 (G0/1)       |
| R1_OSLO ↔ RR2_BGO                 | 192.0.2.10/31    | .10 (G0/5)      | .11 (G0/3)      |
| R2_BGO ↔ RR1_OSLO                 | 192.0.2.12/31    | .12 (G0/5)      | .13 (G0/3)      |
| R2_BGO ↔ RR2_BGO                  | 192.0.2.14/31    | .14 (G0/4)      | .15 (G0/1)      |
| RR1_OSLO ↔ CORE1_OSLO             | 192.0.2.16/31    | .16 (G0/2)      | .17 (G0/1)      |
| RR1_OSLO ↔ CORE2_BGO              | 192.0.2.18/31    | .18 (G0/4)      | .19 (G0/2)      |
| RR2_BGO ↔ CORE1_OSLO              | 192.0.2.20/31    | .20 (G0/4)      | .21 (G0/2)      |
| RR2_BGO ↔ CORE2_BGO               | 192.0.2.22/31    | .22 (G0/2)      | .23 (G0/1)      |
| RR1_OSLO ↔ RR2_BGO                | 192.0.2.24/31    | .24 (G0/5)      | .25 (G0/5)      |
| CORE1_OSLO ↔ PEER1_OSLO           | 192.0.2.26/31    | .26 (G0/3)      | .27 (G/1)       |
| CORE2_BGO ↔ PEER2_BGO             | 192.0.2.28/31    | .28 (G0/3)      | .29 (G/1)       |
| CUST1_OSLO ↔ R1_OSLO              | 198.51.100.0/31  | .0 (G0/3)       | .1 (G0/1)       |
| CUST1_OSLO ↔ CUST_CORE_OSLO       | 198.51.100.2/31  | .2 (G0/1)       | .3 (G0/2)       |
| CUST2_BGO ↔ R2_BGO                | 198.51.100.4/31  | .4 (G0/3)       | .5 (G0/1)       |
| CORE1_OSLO ↔ CORE2_BGO            | 198.51.100.6/31  | .6 (G0/4)       | .7 (G0/4)       |
| CUST2_BGO ↔ CUST_CORE_OSLO        | 198.51.100.8/31  | .8 (G0/1)       | .9 (G0/3)       |

---

## Notes

* All loopbacks are /32 (IPv4) and /128 (IPv6)
* All P2P links use /31 where supported
* Global SRGB range is 16000–23999; node SID = base + router ID