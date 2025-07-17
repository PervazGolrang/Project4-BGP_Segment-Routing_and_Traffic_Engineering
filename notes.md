# Lab Notes

Personal notes and lessons learned from implementing the BGP Segment Routing and Traffic Engineering lab.

---

## Technical Issues Encountered

### Cat8000v License Activation
I had to activate the license on all devices to get all SR features working. By default, the network images do not run a license when starting up. For this lab, you only need the **Network Advantage** license, which includes the necessary advanced routing feautures like BGP, SR, and TE.

You can enable **Network Premier** license, however, it only adds unnecessary features like SD-WAN, QoS, advanced security, analytics, and other stuff not needed for this lab. I still activated the **Network Premiere** license for this project. Just as a security measure if Cisco's documentation was outdated in some aspects. 

Avoid the **Network Essentials**, as you would not have the ability to use BGP, SR/TE, or full OSPFv3 for dual-stack deployment.

You need to activate the license on each router, save to flash, then reboot. Since there are 13 devices, you can use the bootstrap config injection in CML on the `iosx3_config.txt`. 

```bash
license boot level network-premier addon dna-premier
!
hostname <router>
!
end
```

Do note that you do not need to reload, as when CML starts a node, it's essentially a fresh boot. Also, due to a bug that Cisco is continuing to neglect for half-a-decade, if you write `show license summary`, it will show that no license is active. The license is active, you can confirm by activating mpls. A non-activated license would display an error that `mpls ...` is an invalid command.

### CML Performance Tuning
To reduce memory by utilizing deduplication, EVE-NG by default, comes with UKSM, however, CML does not have UKSM or KSM installed by default. Installing and configuring UKSM is far too complex, however, to install and enable KSM is much simpler:

Install the KSM Tuned daemon:
```bash
sudo apt update
sudo apt install ksmtuned
```

Edit the config:
```bash
sudo nano /etc/ksmtuned.conf
```

Change/add these settings for aggressiv deduplication:
```bash
KSM_MONITOR_INTERVAL=20         # How often in seconds KSM runs a check, default is 60s (too long)
KSM_SLEEP_MSEC=10               # Sleep between scan, default is 20ms (a bit conservative)
KSM_NPAGES_MAX=3000             # Maximum pages to scan, default 1250 is a bit low
KSM_THRES_COEF=70               # Activates KSM at 30% RAM usage, default is 20 (far too late)
```

Save with **CTRL+X**, then restart the daemon and enable on boot:
```bash
sudo systemctl restart ksmtuned # Restarts
sudo systemctl enable ksmtuned  # Enables on boot
```

### BGP Route Reflector Configuration
Initially I forgot about cluster-ID configuration and peer-groups when setting up route reflectors. After reading Cisco's SPCORE OCG and white papers of CCNP SP materials, I realized the importance of:
- Proper cluster-ID assignment to prevent routing loops
- Using peer-groups for cleaner configuration management
- Understanding RR client relationships

### eBGP Multihop Issue
Originally I configured `ebgp-multihop 1` but then realized it wasn't needed since all eBGP peers in Step 03 are directly connected via /31 links. This setting:
- Overrides default BGP TTL protection of TTL = 1
- Weakens security by accepting BGP sessions from more than 1 hop away
- Creates risk on public peering and internet transit sessions
- Only needed for peering over loopbacks or via intermediate routes

### BGP Community "Additive" Keyword
Critical discovery: without the `additive` keyword, route communities get replaced instead of appended. This caused major policy issues until I figured out:

```bash
set community 65001:100 additive    # Appends to existing communities
set community 65001:100             # Replaces all existing communities
```

This concept was explained in the Cisco SPCORE OCG.

### Using .0 in /31 Subnets
Using .0 as base for /31 subnets felt weird and several sources mentioned best practice is to avoid .0 and .255 in addressing schemes. Should consider starting /31 ranges from .2/.3 instead. I refused to change the topology of this lab and rewriting, so I'll remember this for future projects.

---

## Learning Discoveries

### Bogon Filtering
Learned the term "bogon filtering", apparently it's supposed to block IP addresses that shouldn't appear on the public internet, like private, reserved, or unallocated IP ranges. In the ENCOR OCG, this concept was glossed over, and they did not call it "Bogon Filtering", but just "route filtering".

### Cisco "Weight" Attribute
I refuse to use BGP "weight" attribute since it's Cisco-only and doesn't exist in other vendor implementations. I prefer standard BGP attributes for a vendor-neutral design.

### Adjacency SIDs
Decided to remove manual Adjacency-SID configuration from the lab. After reading more about TE, found that manual Adj-SIDs are rarely used in real deployments, as OSPF dynamic allocation is usually sufficient, and it got too complex.

---

## Project Complexity Realizations

### SR-TE Complexity
My biggest regret was thinking SR-TE would be "simple basics", as it is a small lab, but it turned out it was much more complex than mentioned in the Cisco OCGs. Especially the interaction between:
- SR-MPLS label operations
- Traffic engineering policies  
- BGP color communities
- Binding-SID allocation
- Candidate-path selection

This required several attempts and re-writes to get working correctly due to my limited Service Provider configuration knowledge.

### Monitoring Complexity
Originally I tried to implement complex telemetry streaming for Step 05 using:
- XPath filters
- YANG-push subscriptions
- gRPC protocol
- Google Protocol Buffers

Caused too many failures and complexities, and this felt like studying for the CCNP SP, which was outside the lab scope. I've simplified it to basic policy monitoring instead.

---

## Network Design Decisions

Removed complex telemetry and focused on basic SR-TE policies that demonstrate the concepts without getting lost in vendor-specific monitoring implementations.

---

## Wireshark Analysis
Compared to my previous projects, this one I planned to use Wireshark to measure actual latency differences and packets between:
- IGP shortest path routing
- SR-TE premium paths (Color 100)
- SR-TE standard paths (Color 200)

I did guess that I may see additional delta delay due to SSD/CPU performance in the lab environment compared to actual production hardware.

---

## Overall Lab Reflection

This lab pushed me way beyond basic BGP configuration into real Service Provider territory. The complexity of integrating:
- Multi-site BGP design with route reflectors
- Segment Routing MPLS implementation
- Traffic Engineering policies
- Business relationship enforcement

This made me appreciate the depth of SP networking. While challenging, the hands-on experience with these technologies provided insights you can't get from just reading documentation, or passing some exam.

The multiple configuration iterations and troubleshooting sessions were frustrating but ultimately valuable for understanding how these protocols actually work together in practice.