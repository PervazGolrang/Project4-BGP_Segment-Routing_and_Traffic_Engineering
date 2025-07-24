# Lab Notes

These are personal notes, reflections and lessons learned from implementing the BGP Segment Routing and Traffic Engineering lab. 

---

## Platform Journey

I researched a lot before starting this lab, I later found out it was not enough. I wrote the configuration steps before setting everything up, by following documentation and my past experience of the network devices. I started with Cat8000v as my **core** network devices since my research showed it was far more feature-rich than CSR1000v. However, halfway-through implementing SR, I discovered that the Cat8000v has severely limited SR-TE support; basic color/candidate-path functionality, with no proper segment lists. I was forced to switch to IOS XRv, which is a Service Provider router with a boot time of 15 minutes plus another 20 minutes for KSM deduplication.

Switching to IOS XRv caused a complete rewrite of all the configurations, from IOS-XE to IOS-XR syntax. There were several problems because Cisco changed names from some logic, e.g. Route-maps became route-policies, prefix-lists became prefix-sets, and a completely different BGP and OSPF address-family structure. Every small difference caused extra delays as I essentially had to learn a new operating system.

The final lab topology ended up being **IOS XRv** for the **six core devices** (R1, R2, RR1, RR3, CORE1, CORE2) and **IOSv** for the external devices (ISPs, peers, customers). IOSv was good enough since I only needed basic BGP functinality. I avoided BGP `address-family ipv4` due to several complaints online said it was buggy.

---

## Technical Discoveries

### eBGP Multihop Confusion

I started the eBGP configurations with `ebgp-multihop 1` on all eBGP sessions, I later realized that it wasn't needed since all eBGP peers are directly conected via /31 links. It would lead to a weakened security since its accepting BGP sessions from more than one hop away. I should focus more on more BGP projects.

### BGP Community "Additive" Keyword

This concept is something I neglected in my CCNP Enterprise studies. It caused about an hour of debugging, onyly to figure out I didn't include the `additive` keyword. Without the keyword, the route communities would get replaced instead of appended.

---

## Performance and Infrastructure

### CML Memory Optimization

CML doesn't include KSM (Kernel Same-page Merging) by default, as compared to EVE-NG (comes with optimized UKSM). Installing and configuring UKSM was far too complex, so I installed KSM, which worked similarly, albeit less efficient, however, it was still a significantly improvement for memory deduplication. Manually installing and configuring KSM was simple:

Install the KSM Tuned daemon:
```bash
sudo apt update
sudo apt install ksmtuned
```

Edit the config:
```bash
sudo nano /etc/ksmtuned.conf
```
Change these settings for more aggressive deduplication settings. This change reduces memory deduplication completion from 50 minutes to 20 minutes:
```bash
KSM_MONITOR_INTERVAL=10         # How often in seconds KSM runs a check, default is 60s (too long)
KSM_SLEEP_MSEC=20               # Sleep between scan, default is 10ms (a bit conservative)
KSM_NPAGES_MAX=10000            # Maximum pages to scan, default is 1250 (very low)
KSM_THRES_COEF=70               # Activates KSM at 30% RAM usage, default is 20 (far too late)
```

Save with **CTRL+X**, then restart the daemon and enable on boot:
```bash
sudo systemctl restart ksmtuned # Restarts
sudo systemctl enable ksmtuned  # Enables on boot
```

---

### Design Decisions

### Cisco "Weight" Attribute

I refuse to use BGP `weight` attribute since it's Cisco-only and doesn't exist in other vendor implementations. I prefer standard BGP attributes for a vendor-neutral design.

### Adjacency SIDs
I decided to remove manual Adjacency-SID configuration from the lab. After reading more about TE, found that manual Adj-SIDs are rarely used in real deployments since OSPF dynamic allocation is usually sufficient.

### Addressing Scheme
Using `.0` as base for /31 subnets felt strange, and I later found out that it is best practice to avoid `.0` and `.255` in addressing schemes. I should have started /31 ranges from `.1/.2` instead, but I didn't want to rewrite the entire topology, again.

---

## Project Complexity

### SR-TE Underestimation
My biggest mistake was thinking SR-TE would be "simple basics" for a small lab. It turned out to be much more complex than expected, especially the interactions between:
- SR-MPLS label operations
- Traffic engineering policies
- BGP color communities
- Candidate-path selection

This required multiple complete rewrites to get working correctly, including a new router OS.

---

## Overall Reflection

This lab pushed me. The complexity of integrating multi-site BGP design, Segment Routing MPLS, traffic engineering policies, and business relationship enforcement made me appreciate how deep SP networking actually goes, and how to fully avoid it until I have more practical experience.

The project took significantly longer than expected compared to my other projects, mainly due to the platform migration, constant rewrites, and SR-TE complexity underestimation.