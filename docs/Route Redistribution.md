# Route Redistribution

Routing redistribution can be used when there is more than one routing protocol used in an AS, and to achieve end-to-end reachability, you can use redistribution to exchange full routing info.

Often, the transition between routing protocols as the environment grows is not fast. 2 or more routing protocols may exist for a long time. 

Different protocols have different requirements and limitations when redistributing, so it is important to be aware of these and configure them carefully.

The main design considerations:

- One or two way redistribution?
- One or multiple redistribution points?
- Impact of external routes on a routing protocol

## The Need for Redistribution

Over time, a network becomes more complex as different network groups adjust it based on factors such as political borders, geographical borders, and m&a.

Routing challenges stemming from differences in the environment:

- different devices/vendors
- branches, hubs, core networks, PoP design
- Different regions/countries
- Multiple separate companies
- M&A causing different entities to smash together

Multiple routing protocols can exist in some example scenarios:

- When migrating from older IGP to new IGP, multiple redistribution boundaries may exist until new IGP completely displaces the old one. Similiarly, this applies for mergers between companies.
- Depending on company, some departments may not upgrade their routers to support new protocols etc
- Mixed vendor environments. EIGRP on Cisco, OSPF on others.
- Modern SP environments may also have colocation, cloud networks where a different set of devices and technology exists


## Route Redistribution Characteristics

Tthe process of using a routing protocol to advertise routes that are learned by the usual means of learning routes, such as by another routing protocol, static routes, or directly connected routes.

- Possible redistribution sources:
    - Other routing protocols, static routes, connected routes
- Internal loop prevention mechanisms
    - Only routes that are actually in the routing table will be redistributed

## Route Redistribution Seed Metrics

- The seed metric or redistributed metric is the starting cost for routes that are redistributed from other protocols. 
- Each protocol behaves differently, some vary depending on protocols

RIP and EIGRP will not redistribute unless you configure a metric (seed it). It will be 0 by default, which is infinity hops, which is unreachable.

For IS-IS, the metric of 0 will be the lowest cost, while in OSPF, it will be 20 (E2).

For BGP, it will copy the originating routing protocols metric as the MED attribute value (metric), which is then propagated throughout.

## Redistribution strategies

Redistribution can bring with it many challenges, and you should approach it carefully to not cause additional issues when trying to solve business needs. Routing across your org should be stable and efficient.

Some of these challenges:

- Suboptimal routing
- Inconsistent convergence timers
- Routing update loops: loop prevent methods do not span protocols (such as iBGP split horizon)
- Route flapping: redistributed routes can preempt each other
- Intermittent routing loops: then when there is route flaps, possibility for loops

### One-Point Route Redistribution

- Route redistribution is only done at one point in the network, and it can either be one-way or two-way
    - If there is one-way, then there should be a default or static route for the other routing protocl to ensure two-way comms
- The advantage of one-point is that there is no risk of routing loops

### Multipoint Route Redistribution

- Same as earlier, except in multiple places and options for both one-way and two-way
- If the receiving routing protocol supports different administrative distances for internal and external routes, it can work better
    - Another common method is to tag routes reditributed from one routing protocol to another, then use route-maps or RPL to prevent re-redistribution of routes at the other redistribution points

### Solutions to route distributions

- Manipulate routing metrics or administrative distance
- Filter redistributed routes to ensure only what you want is being redistributed
- Use tags to simplify identification of redistributed routes
- Summarize redistributed routes when possible
- Consider using a default route to stick with one-way redistribution

- Be wary of administrative distance when doing redistribution
    - For example, if you have a connected route, and you redistribute this on R1 into EIGRP. This will be an EIGRP-external route with 170 AD, but if this is redistributed into OSPF, it will be 110 AD. Possibly this could result in suboptimal routing depending on where the routing decisions are made on the network.

## Configuring Route Redistribution

- `redistribute` command is entered under the importing routing protocol
- You mainly will be concerned with:

    - Source routing protocol (and maybe process)
    - Protocol-specific filters
    - Setting of a default metric
    - Tagging redistributed routes
    - Filtering and modifying routes via `route-maps` and `route-policy`

### OSPF Redistribution

- Use the `subnet`s` option to ensure all routes are redistributed
- Use the `metric-type 1` option to ensure LSA type 5 External type 1 is used so that the cost will be updated and considered throughout the OSPF domain, otherwise link costs will never be appended when propogating the route across the network

``` py title="IOS XE"
router# conf t
router(config)# router ospf 1
router(config-router)# redistribute eigrp 1 subnets metric-type 1 metric 100000 tag 100
```

``` py title="IOS XR"
RP/0/RP0/CPU0:router# conf t
RP/0/RP0/CPU0:router(config)# router ospf 1
RP/0/RP0/CPU0:router(config-ospf)# redistribute eigrp 1 subnets metric-type 1 metric 100000 tag 101
```

### Redistribution into IS-IS

``` py title="IOS XE"
router# conf t
router(config)# router isis 1
router(config-router)# redistribute connected metric 1000 route-map ISIS-Redistribute-Conn
```

``` py title="IOS XR"
RP/0/RP0/CPU0:router# conf t
RP/0/RP0/CPU0:router(config)# router isis 1
RP/0/RP0/CPU0:router(config-isis)# address-family ipv4 unicast
RP/0/RP0/CPU0:router(config-isis-af)# redistribute connected metric 1000 route-policy ISIS-Redistribute-Conn
```

### Redistribution into BGP

- Basically... the exact same

``` py title="IOS XE"
router# conf t
router(config)# router bgp 65001
router(config-router)# redistribute ospf 1 match internal route-map BGP-Redistribute-OSPF
```

``` py title="IOS XR"
RP/0/RP0/CPU0:router# conf t
RP/0/RP0/CPU0:router(config)# router bgp 65002
RP/0/RP0/CPU0:router(config-bgp)# address-family ipv4 unicast
RP/0/RP0/CPU0:router(config-bgp-af)# redistribute ospf 1 match internal route-policy BGP-Redistribute-OSPF
```

## Administrative Distance 

| Routing Protocol | Default AD |
| ----------- | ----------- |
| Connected Route | 0 |
| Static Route | 1 |
| EIGRP summary route | 5 |
| EBGP | 20 |
| Internal EIGRP | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| External EIGRP | 170 |
| iBGP | 200 |


**Modifying OSPF AD**

- Default is 110, and you can modify this with global OSPF command `distance`, or with `distance ospf external xx inter-area yy intra-area zz` to modify the external, inter-area, and intra-area routes more specifically. 
- The command is the same on IOS-XE and IOS-XR
- More selective AD can help with multi-point multi-way redistribution

**Modifying IS-IS AD**

- Default is 115, and this is mod'd with `distance` in global `router isis` mode on IOS-XE and for IOS-XR, under the `address-family ipv4 unicast` config mode.

**Modifying BGP AD**

- Default is 20 for EBGP routes, 200 for iBGP routes
- Use the command `distance bgp $external $internal $local_routes`, like `distance bgp 20 80 80` under global router bgp mode in both IOS-XE/XR 

- Great care should be taken to not advertise BGP next-hops through BGP, which would cause BGP neighborships to flap!

## Routing Loop Prevention Strategies

If you do not take any precautions when doing two-way multi-point redistribution, then there will be a routing loop.

1. Tag redistributed routes
2. Then filter out tagged routes upon redistribution

Let's look at an example with BGP and OSPF two-way multi-point redistribution.

On the two routers, we have BGP and OSPF. We'll create a route-map to deny routes with matching BGP community 10, then set tag 10 on routes redistributed into OSPF from BGP.

``` py title="IOS XE"
router# conf t
router(config)# ip community-list 10 permit 65001:10
router(config)# route-map FilterAndTagBGP deny 10
router(config-router-map)# match community 10
router(config-router-map)# exit
router(config)# route-map FilterAndTagBGP permit 100
router(config-router-map)# set tag 10
router(config-router-map)# exit
router(config)# router ospf 1
router(config-router)# redistribute bgp 65001 subnets metric-type 1 route-map FilterAndTagBGP
```


``` py title="IOS XR"
RP/0/RP0/CPU0:router# conf t
RP/0/RP0/CPU0:router (config-rpl)# route-policy FilterAndTagOSPF
RP/0/RP0/CPU0:router (config-rpl)# if tag eq 10 then
RP/0/RP0/CPU0:router (config-rpl)#   drop
RP/0/RP0/CPU0:router (config-rpl)# else
RP/0/RP0/CPU0:router (config-rpl)#   set community (65001:10)
RP/0/RP0/CPU0:router (config-rpl)#   pass
RP/0/RP0/CPU0:router (config-rpl)# endif
RP/0/RP0/CPU0:router (config-rpl)# end-policy
RP/0/RP0/CPU0:router (config)# router bgp 65001
RP/0/RP0/CPU0:router (config-bgp)# address-family ipv4 unicast
RP/0/RP0/CPU0:router (config-bgp-af)# redistribute ospf 1 route-policy FilterAndTagOSPF
```

