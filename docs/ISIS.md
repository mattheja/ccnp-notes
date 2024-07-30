# IS-IS

## Interlevel Routing and IS-IS

- Overall, IS-IS is a similiar link-state routing protocol with similiar issues and limitations
- When you have a single-level IS, the primary limit is the number of nodes
- Multilevel allows for less nodes for a smaller LSDB to scale

***NSAP Examples***

The NSAP 49.0001.aaaa.bbbb.cccc.00 consists of the following:

- For IS-IS:
    - Area = 49.0001
    - System ID = aaaa.bbbb.cccc
    - N-selector = 00

- For ISO-IGRP:
    - Domain = 49
    - Area = 0001
    - System ID = aaaa.bbbb.cccc
    - N-selector = ignored by ISO-IGRP

The NSAP 39.0f01.0002.0000.0c00.1111.00 consists of the following:

- For IS-IS:
    - Area = 39.0f01.0002
    - System ID = 0000.0c00.1111
    - N-selector = 00
- For ISO-IGRP:
    - Domain = 39.0f01
    - Area = 0002
    - System ID = 0000.0c00.1111
    - N-selector = ignored by ISO-IGRP


## Multilevel IS-IS

**Three types of routers**:

1. <ins>Level 1 (L1)</ins>: only learns about paths within the area they connect to (intra-area only) - edge/branch

2. <ins>Level 2 (L2)</ins>: learns about paths between area (inter-area) -- provider core

3. <ins>Level 1-2 (L1-2)</ins>: learn about paths both within and between areas. Similiar to OSPF ABR's.


Most notably, area boundaries fall on links. A router belongs in ONE level only.

Adjancies are negotiated based on whether they are in the same area or a different area -> L1, L2, or L1-2 neighborship

By default, IS-IS areas are considered "totally stubby" by default, but you can leak L2 routes into L1

### Configuring Multilevel IS-IS on IOS-XR
- Interfaces must be enabled for IS-IS, and the IS-type is Level1-2 by default. 
- L2 used in core where there are only L2 link states exchanged
``` py title="IOS XR - P1 - net 49.0000 = area id (0)"
router# conf t
router(config)# router isis 1
router(config-isis)# net 49.0000.0001.0001.1001.00
router(config-isis)# is-type level-2-only
router(config-isis)# interface gi0/0/5/2
router(config-isis-if)# ip router isis
router(config-isis-if)# isis circuit-type level-2-only
```
``` py title="IOS XR - PE router - net 49.0002 = area id (2)"
router# conf t
router(config)# router isis 1
router(config-isis)# net 49.0002.0001.0001.1001.00
router(config-isis)# is-type level-1-2
router(config-isis)# interface gi0/0/0/1
router(config-isis-if)# address-family ipv4 unicast
router(config-isis-if)# isis circuit-type level-2-only
router(config-isis)# interface gi0/0/0/2
router(config-isis-if)# address-family ipv4 unicast
router(config-isis-if)# isis circuit-type level-1-2
router(config-isis)# interface gi0/0/0/3
router(config-isis-if)# address-family ipv4 unicast
router(config-isis-if)# isis circuit-type level-1
```
- On the different interfaces, there will be different adjancies negotiated. 
    - L2-only used in the core where only L2 LSP's are exchanged.
    - L1-2 used with other L1-2 routers where both L1 and L2 LSP's are exchanged
    - L1-only used with other L1-only routers to exchange only L1 LSP's

### Configuring Multilevel IS-IS on IOS-XE
- Generally it is the same as IOS-XR. Interfaces must be specifically enabled for IS-IS with the type selected

``` py title="IOS XE - P1 - net 49.0000 = area id (0)"
router# conf t
router(config)# router isis 1
router(config-router)# net 49.0000.0001.0001.1001.00
router(config-router)# is-type level-2-only
router(config-router)# exit
router(config)# interface Gi0/1/5
router(config-if)# ip router isis
router(config-if)# isis circuit-type level-2-only
```
``` py title="IOS XE - PE - net 49.0002 = area id (2)"
router# conf t
router(config)# router isis 1
router(config-router)# net 49.0002.0001.0001.1001.00
router(config-router)# is-type level-1-2
router(config-router)# exit
router(config)# interface Gi0/0/1
router(config-if)# ip router isis
router(config-if)# isis circuit-type level-2-only
router(config-if)# exit
router(config)# interface Gi0/0/2
router(config-if)# ip router isis
router(config-if)# isis circuit-type level-1-2
router(config-if)# exit
router(config)# interface Gi0/0/3
router(config-if)# ip router isis
router(config-if)# isis circuit-type level-1
```

## IS-IS Route Leaking
- By default, routes are leaked from L1 to L2, but IS-IS will block L2 -> L1
- You usually want to ensure that BGP next-hop addresses are reachable, so leaking these routes for reachability is important. 

``` py title="IOS XE"
router# conf t
router(config)# router isis 1
router(config-router)# redistribute isis ip level-2 into level-1 route-map RM-ISIS
```
``` py title="IOS XR"
router# conf t
router(config)# router isis 1
router(config-isis)# address-family ipv4 unicast
router(config-isis-af)# propagate level 2 into level 1 route-policy RP-ISIS
```

## Prefix Suppression
- Option to further limit the size of the LSDB. For example, to only advertise edge networks and BGP next-hop addresses. 
- This can reduce the convergence time
- Commonly you would exclude transport links
- If you configure interfaces to be passive in IS-IS, then only those interfaces with be propagated into IS-IS

- There are two ways to configure prefix suppression:
    1. On a per-interface basis
    2. On a per-router basis

``` py title="IOS XE"
L1_L2-router# conf t
L1_L2-router(config)# router isis 1
L1_L2-router(config-router)# advertise passive-only
OR
L1_L2-router(config)# interface Gi0/2/1
L1_L2-router(config-if)# no isis advertise-prefix
```
``` py title="IOS XR"
router# conf t
router(config)# router isis 1
router(config-isis)# address-family ipv4 unicast
router(config-isis-af)# advertise passive-only
OR
router(config-isis)# interface Gi0/0/1/2
router(config-isis-if)# suppressed
```

- Suppression = stops advertisement of PDU's over these links and also configures the IS-IS interface not to advertise its IP network to neighbors 

## IS-IS Interlevel Routing Guidelines
- Place a L1-2 border router between the core and POP uplink routers
- Use route leaking to ensure PE loopbacks are reachable and do not summarize
- Use prefix suppression to minimize LSDB size


## IS-IS Path Selection
- Similiar to OSPF, in a way.

1. **Level**
    - L1 route has highest preference
    - L2 route as alternative
2. **Paths of same level**
    - Internal route has highest preference
    - External route is alternative
3. **For routes of the same level and origin**:
    - Lowest metric
    - All interfaces default to a metric of 10

### Configuring IS-IS Metrics
- The IS-IS metric is similiar to OSPF, but there is no automatic mechanism for this to happen based on link bw. 
- The default original metric is 1-63, while there is a wide configurable range between 1-16777214 after enabling `metric-style wide`

``` py title="IOS XE"
L1_L2-router# conf t
L1_L2-router(config)# router isis 1
L1_L2-router(config-router)# metric-style wide
L1_L2-router(config-router)# metric 15625000
L1_L2-router(config-router)# exit
L1_L2-router(config)# interface Gi0/5/1
L1_L2-router(config-if)# isis metric 1000
```
``` py title="IOS XR"
L1_L2-router# conf t
L1_L2-router(config)# router isis 1
L1_L2-router(config-isis)# address-family ipv4 unicast
L1_L2-router(config-isis-af)# metric-style wide
L1_L2-router(config-isis-af)# metric 15625000
L1_L2-router(config-isis-af)# exit
L1_L2-router(config-isis)# interface Gi0/0/1/2
L1_L2-router(config-isis-if)# metric 1000
```

### IS-IS Path Selection Guidelines
- Use wide metric everywhere
- Change the default metric to a high value on slow interfaces from attracting traffic
    - Default is 10, lowest = best
- Design a metric plan since there is no automatic metric calc in IS-IS
- Consider borrowing OSPF formula for link speeds and metrics

## IS-IS Summarization
- Solves large routing tables and frequent link-state flooding throughout the AS
- Reducing the size of database improves scalability
- By default, when a route flaps or disappears in one area, routers in other areas and levels get involved in SPF calculation
- Like OSPF, you can summarize routes but this is done on the Level boundaries in IS-IS, and they can be used with external redistributed routes

``` py title="IOS XE"
router# conf t
router(config)# router isis 1
router(config-router)# redistribute ospf 1 level-1-2
router(config-router)# summary-address 10.1.0.0 255.255.0.0 level-1
router(config-router)# summary-address 10.2.0.0 255.255.0.0 level-2
router(config-router)# summary-address 10.3.0.0 255.255.0.0 level-1-2
```

``` py title="IOS XR"
router# conf t
router(config)# router isis 1
router(config-isis)# address-family ipv4 unicast
router(config-isis-af)# redistribute ospf 1 level-1
router(config-isis-af)# redistribute ospf 1 level-2
router(config-isis-af)# summary-prefix 10.1.0.0/16 level-1 
router(config-isis-af)# summary-prefix 10.2.0.0/16 level-2
router(config-isis-af)# summary-prefix 10.3.0.0/16 level-1-2
```

- This is a weird example, but it shows `summary-prefix` is used for IOS-XR and `summary-address` is used for IOS-XE with the different summary options:
    - **level-1**: only routes distributed into Level 1 are summarized
    - **level-1-2**: summary routes applied when redistributed into Level 1 and 2 IS-IS, and when Level 2 "leaks" or advertises Level 1 routes as reachable
    - **level-2**: routes learned by Level 1 routing are summarized into Level 2 backbone; redistributed external routes will also be summarized into Level 2 too.
- Redueces the size of LSP's and thus the LSDB
- More stable because a summary advertisement depends on many specific routes, and as long as one of these specific routes is available, the summary route won't flap.
- Drawback is possibly less information to calculate the most optimal route in large topologies

## IS-IS Fast Convergence
- By default, it can take IS-IS more than 30 seconds to converge upon link or device failure
- There are some techniques to improve this:
    1. Tune IS-IS timers 
    2. Cisco NSF
    3. Cisco NSR
    4. BFD

### Cisco NSF, NSR
- More or less the same as NSF for OSPF
- Feature for IOS-XR to allow the forwarding of data packets to continue while protocol information is restored following a failover event between RP's on a NSF-capable router
- The standby RP during failover must relearn neighbor relationships and reacquire contents of LSDB's
- NSF-capable and NSF-aware neighbors do not flap during a RP failover using NSF
- NSR allows for RP failover, process restarts, and in-service upgrades to be invisible to peers.
- Using NSR alleviates the requirements for NSF and IETF graceful restart procotol extensions

*from a LLM about graceful restart protocol extensions:*

The IETF has defined graceful restart protocol extensions for IS-IS in RFC 5306. These extensions aim to minimize traffic loss and network disruption during a planned or unplanned IS-IS router restart. Here's how the IS-IS graceful restart mechanism works:

1. Graceful Restart Capability Advertisement:
    - Routers advertise their support for graceful restart using the Graceful Restart TLV (Type-Length-Value) in their IS-IS Hello PDUs.
    - The TLV includes information such as the restart duration (how long the restarting router will take to complete the restart) and the restart reason (planned or unplanned).

2. Planned Restart:
    - When a router undergoes a planned restart (e.g., for software upgrade), it sends IS-IS Hello PDUs with the Graceful Restart TLV to its neighbors, indicating its intention to restart.
    - The restarting router's neighbors (helper routers) will continue to advertise the restarting router's routes as if it were still active, using the restart duration advertised in the Graceful Restart TLV.

3. Unplanned Restart:
    - If a router experiences an unplanned restart (e.g., due to a crash), it will not have the opportunity to send the Graceful Restart TLV in advance.
    - Upon restart, the router will send IS-IS Hello PDUs with the Graceful Restart TLV, indicating that it has just restarted and is in the process of recovering its state.

4. Helper Router Behavior:
    - Neighbors of the restarting router (helper routers) will continue to advertise the restarting router's routes for the duration specified in the Graceful Restart TLV.
    - Helper routers will mark the restarting router's routes as "stale" but will continue to forward traffic to those routes.
    - If the restarting router does not re-establish adjacency within the specified restart duration, the helper routers will remove the stale routes.

5. Restarting Router Recovery:
    - During the restart, the restarting router will rebuild its routing table and LSDB (Link State Database) based on the information received from its neighbors.
    - Once the restarting router has rebuilt its state, it will send updated LSPs (Link State PDUs) to its neighbors, indicating that it has completed the restart process.

6. Synchronization and Convergence:
    - Upon receiving the updated LSPs from the restarting router, the helper routers will synchronize their LSDBs with the restarting router.
    - The network will converge, and normal IS-IS routing will resume.

The IS-IS graceful restart mechanism helps minimize traffic disruption during router restarts by allowing the restarting router's neighbors to continue forwarding traffic to the restarting router's routes while it rebuilds its state. This provides a smoother transition and faster convergence compared to a complete network reconvergence.

### Other factors affecting convergence
- Not all underlying physical topologies support fast link failure notifications
- carrier delays could introduce additional delay
- slower timers could yield a more stable environment at the expense of faster convergence

### IS-IS events influencing convergence
| Event | Action | Timers | Default Timer |
| ----------- | ----------- | ----------- | ----------- |
| Local LSP change | Originate LSP and flood | LSP generation timer | 5 seconds |
| LSP db and tree change | Run iSPF/SFP calculation | SFP interval | 10 seconds |
| LSP db but no tree change | Run PRC calculation | PRC interval | 5 seconds |
| Scheduled | Periodic LSP flooding | LSP refresh interval | 900 seconds (15mins) |
| LSP expires | LSP refreshing | Max LSP lifetime | 1200 seconds |
| Pace LSPs (30 LSP/s) | LSP flooding | LSP interval | 33ms |
| LSP interval | Run iSPF/SPF and flood LSP | CPU-related delays trigger SPF early | - |

### LSP Processing and Propagation
ISO 10589 states LSP flooding is limited to 30 LSP's per second
There are design suggestions to improve LSP processing:

1. Reduce the gap to speed up end-to-end flooding
2. Reduce unnecessary control plane traffic to converse CPU for LSP's
3. Reduce the frequency of periodic LSP flooding to reduce link utilization
4. Ignore LSP's with bad checksum instead of purging


Throttling slows convergence, but it prevents "meltdown" that would cause further problems. The goal is to react fast to the first events but, under constant churn, slow down to avoid collapse.


- During convergence, IS-IS throttles the following events:
    - SPF computation
        - IS-IS throttles SPF calculation to prevent excessive CPU
        - When there is a LSP upgrade to trigger SPF calc, IS-IS starts a SFP timer
        - If additional LSP updates are rx'd during this timer, IS-IS will wait until timer completes before running SPF, incorporating all the changes
    - LSP generation
        - when there is an event that triggers generating a new LSP, this starts the generation timer and hold-down timers
        - if additional triggering events IS-IS occur during the hold-time timer, IS-IS will wait for the generation timer to expire before generating a new LSP, incorporating all the changes        
    - Partial route calculation (PRC) computation
        - Similiar as above 
> PRC is the software’s process of calculating routes without performing an shortest path first (SPF) calculation.
> This is possible when the topology of the routing system itself has not changed, but a change is detected in
> the information announced by a particular IS or when it is necessary to attempt to reinstall such routes in the
> Routing Information Base (RIB).


## Carrier Delay
- If a link goes down and up before the carrier delay expires, then the down state event is filtered
- Higher delay = more stability; while setting to 0 delay = everything is detected
- Most environments want low or default carrier delays, but if there tends to be a high frequency of small outages, then it might be prudent to not cause disruptions and trigger re-convergence
- If there are rare but long outages, short delay or 0 delay better

## Configuring IS-IS for Fast Convergence

IOS-XE only example:

Look at [Cisco IOS IP Routing: ISIS Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_isis/command/irs-cr-book.pdf) for additional info on each of these...

``` py title="IOS XE"
router# conf t
router(config)# router isis 1
router(config-router)# bfd all-interfaces
router(config-router)# fast-flood 15  # send 15 LSP's before running SPF, default = 5
router(config-router)# isis lsp-interval 10  # reduces LSP interval, default 33ms
router(config-router)# max-lsp-lifetime 65535  # LSP lifetime up from default 1200s
router(config-router)# lsp-refresh-interval 65000  # up from default 900s
router(config-router)# ignore-lsp-errors
router(config-router)# spf-interval 5 1 20  # default is 10s 5.5s 5500ms    \
router(config-router)# prc-interval 5 1 20  # default is 10s 5.5s 5500ms     |- these 3 are backoff timers
router(config-router)# lsp-gen-interval 5 1 20  # default is 5s 50ms 5000ms /
router(config-router)# ispf level-1-2 60  # incremental SPF calc timer
router(config-router)# no hello padding
router(config)# interface gi1/0/1
router(config-if)# bfd interval 50 min_rx 50 multiplier 5  # fast link detected via BFD
router(config-if)# isis network point-to-point  # disable DIS election
router(config-if)# carrier-delay down msec 0  # immediate down notification
router(config-if)# carrier-delay up 2  # delayed link-up notification
```

Regarding `ispf`:
> When changes to a Type 1 or Type 2 link-state advertisement (LSA) occur in an area, the entire SPT is
> recomputed. In many cases, the entire SPT need not be recomputed because most of the tree remains unchanged.
> Incremental SPF allows the system to recompute only the affected part of the tree. Recomputing only a portion
> of the tree rather than the entire tree results in faster OSPF convergence and saves CPU resources. 

## ISIS Convergence Guidelines
- Set goals and also account for other protocol options like MPLS TE like Fast Reroute 
- Suggestions:
    - Use BFD over faster hellos
    - Consider carrier delay to minimize process times
    - Use p2p mode on core links to avoid DIS election
    - Tune timers to improve process/flooding LSPs


## IS-IS for IPv6
- Multiprotocol by default, supports both IPv4 and IPv6 wiwth multitopology support to allow IS-IS to maintain a set of independent topologies within a single area or domainq
    - Not all routers inside the IS-IS domain need to all be dual-stack. The topologies are separate, SPF is calculated for each independently.
    - For multitopology support, all routers in the domain should be multitopology enabled

- On IOS-XE/XR, one must use `metric-style wide` to configure new-style TLV's for IPv6 info in LSP's when using multitopology 
- when you migrate from pure-IPv4 to dual-stack, there is the `adjacency-check disable` (IOSXE) or `no adjacency-check` configuration to disable consistency checks on hello packets to allow a router running IS-IS for both IPv4 and IPv6 to form an adjacency with a router running IS-IS for IPv4 or IPv6 only.

### Configuring IS-IS for IPv6

``` py title="IOS XE"
router# conf t
router(config)# router isis 1
router(config-router)# metrics-style wide
router(config-router)# address-family ipv6 unicast
router(config-router)# exit
router(config)# int Gi1/0/1
router(config-if)# ipv6 enable
router(config-if)# ipv6 router isis
```

``` py title="IOS XR"
router# conf t
router(config)# router isis 1
router(config-isis)# address-family ipv6 unicast
router(config-isis-af)# single-topology         # IOS-XR is multi-topology by default
router(config-isis-af)# exit
router(config-isis)# int Te1/0/3 
router(config-isis-if)# address-family ipv6 unicast
router(config-isis-if)# exit
router(config-isis)# int Te1/0/4
router(config-isis-if)# address-family ipv6 unicast
```

### Route Summarization for IPv6
- Both IOSXE/XR use `summary-prefix` for summarization of IPv6 prefixes, everything else for route summarization is the same as in IPv4 described earlier

## IS-IS v OSPF Comparisons 
- Both are link-state protocols with similiar characteristics (different terminology sometimes), but there are large design differences:
    - OSPF, different interfaces can be in different areas, while in IS-IS, the router itself is assigned into an area. 
    - IS-IS is not constrained to using a backbone area, so there can be more flexibility to modify topology

- IS-IS has a heirarchy of Level-1, Level-2, and L1-L2 routers, and the area borders are on links (as the routers themselves are only in one area)
- OSPF can produce many small LSA's, while IS-IS updates can be grouped and sent as one LSP
    - This can add up to be a large "pro" for why IS-IS can be more scalable in single-area designs over OSPF

- OSPF runs on top of IP, while IS-IS uses Connectionless Network Service (CLNS) that is a L2 protocol. 
- IS-IS is also more efficient than OSPF in the use of CPU resources in how it processes routing updates due to less LSPs

- OSPF schema to add new features with LSA's is challenging due to compatability issues, while IS-IS is easy to extend through the TLV extension

### Comparing OSPF and IS-IS terminology

| Terminology | OSPF | IS-IS |
| ----------- | ----------- | ----------- |
| Node | Router | IS |
| Link | Link | Circuit |
| Multiaccess node | DR | DIS |
| Redundant Multiaccess node | BDR | n/a |
| Routing update | link-state advertisement (LSA) | link-state PDU (LSP) |
| Keepalive | Hello packet | IS-IS Hello (IIH PDU) |
| Area | Area | Subdomain or area |
| Inter-area nodes | ABR | Level 1 Level 2 IS |
| Node identifier | Router ID | System ID |

| Characteristic | OSPF | IS-IS |
| ----------- | ----------- | ----------- |
| Transport protocol | IP | CLNS |
| Node identifier | IPv4 (router ID) | NSAP (system ID) |
| Area boundary | Node: ABR in multiple areas / Link in one area | Link: Level 1 Level 2 router in one area / link in multiple areas |
| Link in two areas/levels | Virtual Link | Level 1 Level 2 link |
| Multiprotocol | No | Yes |
| High availability | DR and BDR | Only one DIS |
| Automatic cost | Yes | No |