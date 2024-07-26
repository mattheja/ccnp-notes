# Multicast

## Multicast Distribution Tree 

High level, multicast traffic is sent and distributed (forwarded) along a Multicast Distribution Tree (MDT). The MDT is the data plane.

A source sends traffic into a MDT and receives join the MDT to receive traffic. 

In a MDT, a source sends traffic, and the first hop router (FHR) has a special role in creating the MDT. The sender does not use any control plane signaling, it just sends traffic. Although, sources can "subscribe" to a group (multicast address), and the source will send to that.

Internet Group Management Protocol (IGMP) is used by receivers. On the network, PIM, and across multidomain networks, MP-BGP and Multicast Source Discovery Protocol (MSDP) are used.

<br>
Below we will cover the different MDT's.

### SSM Distribution Tree (S,G)

Source-specific Multicast (SSM) is commonly used for applications such as video or audio streaming, where a single source sends to a group of receivers. The receivers join explicitly to this single source. 

Receivers join a specific group, referred to as (Source,Group) or (S,G). This (S,G) state must be kept in the network, and thus impacts resource consumption on routers. Every new unique (S,G) entry will lead to more state on the network.

SSM may not scale well in all cases. If a receiver requests to join multiple sources, it will create (S2,G2), (S3,G3), etc. There will be several sources streaming to the same group. Any Source Multicast (ASM) is preferred here to create a shared MDT.
<br>

### ASM Distribution Tree: (*,G)

ASM is commonly used for video conferencing, or applications with several sources in the same group. In this mode, a receiver joins any source in a specific multicast group.

The source is joining ***all*** sources, as it is (*,G). In ASM, all traffic from all sources is first sent to a Rendezvous Point (RP).

Receivers will send a Join request to the RP, and then a shared MDT is created from the RP to the receiver. In some cases, this could result in sub-optimal routing/delay between sources and receivers due to RP.

### Loop Avoidance in MDT

In unicast, the routing protocol checks for loops. In multicast, a different mechanism must be used: **Reverse Path Forwarding (RPF)**.

RPF works in the following way:

- For each incoming multicast packet, the router looks up the source address in the unicast routing table. It looks for which interface it would use, if it were to send a normal unicast packet to that source.
- If the packet was received on the same interface, it is accepted.
- If the packet arrived on any other interface, it is dropped. In this case, it has come from another router which is not directly in the path. Forwarding such a packet would lead to a multicast routing loop. 

<br>
An example would be (S,G) of (192.0.100.1,239.1.1.1) coming into R1 on Gi0/1 interface.

RPF would work like-- unicast routing lookup on 192.0.100.1. R1 RIB shows Gi0/2 for 192.0.100.0/24. RPF **fails** and packets discarded.

### Multicast IP ranges

Multicast traffic is sent to specific multicast addresses. The group G in (S,G) and (*.G) is such a multicast address.

| Multicast Range      | Description                          | Notes |
| ----------- | ------------------------------------ | -------------------- |
| `224.0.0.0 - 224.0.0.255` | Local Network Control Block | Never routed, strictly link local |
| `224.0.1.0 - 224.0.1.255` | Internetwork Control Block | Can be routed, *internet* control block |
| `224.0.2.0 - 224.0.255.255` | AD-HOC Block 1 | Legacy assignments by IANA |
| `224.2.0.0 - 224.2.255.255` | SDP/SAP Block per rfc5771 | For apps that use SDP/SAP |
| `224.3.0.0 - 224.4.255.255` | AD-HOC Block 2 | Legacy assignments by IANA |
| `232.0.0.0 - 232.255.255.255` | Source-Specific Multicast 232/8 | All SSM groups must use this range |
| `233.0.0.0 - 233.251.255.255` | GLOP Block | inter-AS multicast traffic but doesn't work with 4 byte ASN's |
| `233.252.0.0 - 233.255.255.255` | AD-HOC Block 3 | Assignments by IANA for stuff |
| `234.0.0.0 - 234.255.255.255` | Unicast-Prefix-based IPv4 Multicast addresses | rfc6034 |
| `239.0.0.0 - 239.255.255.255` | Organization-Local Scope | Loosely intended to be "private multicast addresses" |

<br>

## IGMP

The IGMP is the protocol between receivers and the last hop routers. It lets receivers subscribe to groups, and when they are done, it lets those receivers stop receiving the traffic later on.

The IGMP is a control plane protocol that allows receivers to signal their interest in specific multicast groups. It runs exclusively between receivers and a last hop router, using link local or multicast addressing.

**IGMPv1**: no longer used. Many drawbacks.

**IGMPv2**: most commonly supported today, but it does not support SSM.

**IGMPv3**: most recent with features + SSM support.

IGMP running on a router is always backwards compatible. A router with v3 can talk to a receiver using v1 or v2.

### IGMPv2

- Receivers can join and leave a group
    - Version 2 membership report (type value 0x16): called **IGMP Join**; used by Receivers to join a multicast group or to respond to a local router's membership query 
    - Version 1 membership report (0x12): used by Receivers for backward compatibility with IGMPv1 
    - Version 2 leave group (0x17): Receivers indicate they want to stop multicast traffic from a group they joined 
<br>
- Queriers (last-hop routers):
    - There are two type of queries, **general membership query** and **group specific query**.
    - If a LHR receives a "leave" message from one of the receivers on the subnet, it will send a general membership query to try and prune but wait for potential membership report(s) from receivers.
    - General membership query (0x11): sent to All-Hosts group address 224.0.0.1 to see if there are any Receivers on attached subnet; group address field is set to 0.0.0.0 
    - Group specific query (0x11): sent in response to a leave group message to group address they requested to leave 

Ohter information, the max response time: max time before sending a response report; typically 10s.

### IGMPv3

Same but more. Allows receivers to:

- Join specific sources, which is required for SSM. "source S1, S2, S3", "all sources except S2".
- Source filtering, can listen to (S1, G) but filter (S2,G)

Backwards compatible with v2 and new fields added to Membership Query and v3 Membership Report.

SSM allows for the creation of source distribution trees without the need for a Rendezvous Point (RP).


<br>

## ASM v SSM 

Multicast has two fundamental architectural methods to create a MDT. In ASM, the network finds the sources, as receivers join (*,G) so receivers join all sources at once in a shared tree. For SSM, the receiver joins an explicit source (MDT) so complexity is lower to deliver packets, especially for inter-domain topologies.

SSM is usually prefered, but there can be more state to manage on the routers. 

### SSM

Receiver driven, so the flow starts from receiver side.

- The receiver learns about the source and group address out-of-band (OOB), from a web page, an application, or similar.
- The app in the receiver then sends an IGMPv3 membership report to the last hop router. In this request, it specifies the specific (S,G) that it wants to receive traffic from. 
- PIM join messages from LHR are sent hop by hop, to "all PIM routers" in a multicast group 224.0.0.13. The next hop is determined by a lookup in the unicast routing table. This method will send PIM join messages hop by hop, until the first hop router is reached. This leads to the optimal path of the source (as defined in the unicast table), while on each router on the way—multicast forwarding state is established.
- The source sends multicast stream to FHR

When a LHR has no more receivers, it sends a PIM Prune message upstream to stop the multicast stream.

### ASM

In ASM a receiver does not join a specific source, but just a group (*,G). So how does the network know what sources to send if it gets no instructions from the receivers?? It finds them using a RP for receivers. 

**Step one**, a PIM join is sent from the LHR to the RP. This builds a "source specific tree" to the RP (intermediate source). 

The process is the same as SSM, the application in the receiver then sends an IGMP membership report (version 2 or version 3) to the last hop router. In this request it specifies the group (*,G) it wants to receive traffic from.

As in SSM, PIM join messages are sent hop by hop, to the "all PIM routers" multicast group 224.0.0.13. In ASM however, the messages are sent toward the RP, because the last hop router does not know the source yet. The next hop is determined by a lookup of the RP in the unicast routing table. This method will send PIM join messages hop by hop, until the RP is reached. This leads to the optimal path to the RP, while on each router on the way—multicast forwarding state is established.

Now the RP has a shared source tree to all receivers. 

**Step two**, a source starts forwarding multicast stream. In ASM, FHR sends the multicast traffic through a unicast tunnel to the RP, using PIM-register messages. This informs the RP about the source.

Next, the RP uses the previously established Shared Tree to forward to receivers.

Now, the sources and receivers are linked, but it could be sub-optimal. Is the route between source - RP - receiver the best? Also, traffic between the source and RP is still in the unicast tunnel.

**Step three**, the RP sends back PIM-register messages to the source. When the FHR has the native multicast state installed, it starts to send multicast in parallel with the unicast traffic. Once the RP receives the multicast traffic, it will signal the FHR to stop the tunnel with a PIM register-stop message. 

Now there are two distinct trees: (1) Source to RP, (2) RP to receiver

**Step four**, the LHR will be made aware of the source addresses as traffic is forwarded to it via step 2 above. The LHR will send direct (S,G) to each source. This will result in a source distribution tree, then PIM will naturelly prune paths no longer used. 


## Intradomain and Interdomain Multicast 

Interdomain multicast routing architecture allows for an administrative and secure separation between domains while allowing for direct multicast between domains.

There are challenges with routing between interdomain RPs, and the signaling and data plane functionality of PIM with RPs can allow receivers and sources to just start sending with any controls. This is a utilization/capacity risk and access risk.

Within a sinle domain, IGMPv2/v3 (between receivers - routers), PIM-SM (between routers) is used for multicast. 

For interdomain, there is:

**MP-BGP**: Multiprotocl-BGP (MP-BGP) supports many address-families in parallel, and it used to advertise prefixes for multicast traffic between domains. It allows for two routing tables, such that multicast traffic can be routed over another link from unicast traffic. It it still a unicast routing table though, but when PIM creates the MDT, it can use the MP-BGP multicast table for the RPF check.

**PIM-SM**: used to create MDTs between routers across domains.

**Multicast Source Discovery Protocol (MDSP)**: used to connect different PIM-SM domains. It allows sources in one PIM-SM domain to be discovered by RPs in other PIM-SM domains. 
    - When a new multicast source starts sending traffic in a PIM-SM domain, the FHR registers the source with its local RP using a PIM Register message.
    - The local RP then uses MSDP to advertise this new multicast source to MSDP peers in other PIM-SM domains.
    - The MSDP peers then inform their local RPs about the new source, allowing them to join the source-specific distribution tree.
    - This way, MSDP enables RPs in different PIM-SM domains to learn about active multicast sources, allowing receivers in one domain to join and receive multicast traffic from sources in other domains.

### More on MSDP

RP operators want to secure and run their own RPs, they do not want to "share" a RP with another domain. So, MSDP allows RPs in different domains to continue working the same for intradomain multicast. 

An RP in a PIM-SM domain has MSDP peering relationships with MSDP-enabled routers in other domains. Each peering relationship occurs over a TCP connection, which is maintained by the underlying routing system. 

MSDP speakers exchange messages called Source Active (SA) messages. When an RP learns about a local active source, typically through a PIM register message, the MSDP process encapsulates the register in an SA message and forwards the information to its peers. The message contains the source and group information for the multicast flow, as well as any encapsulated data. If a neighboring RP has local joiners for the multicast group, the RP installs the (S,G) route, forwards the encapsulated data contained in the SA message, and sends PIM joins back towards the source. 

SA messages contain the following fields:

- Source address of the data source.
- Group address that receives data sent by the source.
- IP address of the RP.


Routing is then done via underlying routing protocol(s), although it is recommended to use MP-BGP, it is not 100% necessary.

### Interdomain Shared MDT creation

- When a source’s first data packet is registered by the first hop router, the RP extracts the data from the packet and forwards it down the shared tree in the PIM domain.
- The RP informs MSDP peers of the new source by sending a Source-Active (SA) message that identifies the source, the recipient group, and the RP’s address or originator ID.
- Upon receiving the SA message, an MSDP peer which is the RP for a multicast tree that includes members interested in the multicast sends a PIM join message (S,G) toward the data source.
- After the RP on another domain joins the PIM Designated Router (DR) in the first domain, multicast data traffic flows natively over the multicast tree to the second domain's RP.
- If the source times out, this process repeats when the source goes active again. 

### Interdomain SSM

Much more simple. There is no requirement for RPs, thus no MDSP. However, routing still has to happen, so MP-BGP is still required to keep the multicast source table separate from the unicast one. And PIM-SM is used to create the MDT.

IPv6 does not use MDSP, since IGMP is a IPv4-only. IPv6 receivers use Multicast Listener Discovery (MLD), which does not require the usage of MDSP.

<br>

## Multicast IP Layer 3 to Layer 2 Mapping

Layer 2 has its own multicast scheme, developed independently from IP multicast at layer 3.

Layer 2 MAC addresses have 6 bytes (48 bits), where 25 bytes are fixed, leaving effectively 23 bits for the Layer 2 multicast address.

IPv4 multicast addresses use 4 bytes (32 bits), of which 4 bits at the beginning are fixed ("1110") to indicate that it is a multicast address, leaving 28 bits for effective addressing.

32 (2^5) IP addresses map into each one of the Ethernet addresses. This does not cause any functional problems, but it may lead to sub-optimal multicast traffic distribution. Several multicast streams may be delivered to an endpoint, even though it only subscribed to one. However, the endpoint would just discard unneeded traffic. Since this happens typically on the switch before the endpoint, there is normally no problem. 

### IGMP Snooping

IGMP snooping sest up Layer 2 multicast forwarding tables to deliver traffic only to ports with at least one interested member, significantly reducing the volume of multicast traffic. IGMP snooping uses the information in IGMP membership report messages to build corresponding information in the forwarding tables to restrict IP multicast traffic at Layer 2.

Without IGMP snooping, a switch will forward a copy of multicast packet to all network interfaces. 

rfc4541 discusses this further.


### PIM Snooping

If there are several routers connected to a L2 switch, that switch does not know which routers are subscribed to the multicast traffic. It is a similiar problem, so a similiar solution was created with PIM snooping.

With PIM snooping the switch interprets all Layer 3 PIM messages. This way it learns which router wants to subscribe to which group and can forward the multicast traffic only to the correct ports.

PIM snooping can be enabled globally on a switch, or per interface. 

`ip pim snooping`

`interface vlan 4090`
`ip pim snooping`


## Multipoint LDP

Label Switch Multicast (LSM) is MPLS technology extensions to support multicast using label encapsulation.

This allows for the setup of point-to-multipoint (P2MP) and multipoint-to-multipoint (MP2MP) Label Switched Paths (LSPs) in MPLS networks. These extensions are also referred to as multipoint LDP. These LSPs can be used for transporting both IPv4 and IPv6 multicast packets, either in the global table or VPN context.    

Just like in standard multicast (outside MPLS), the MPLS core unicast routing and switching is also required to support multicast traffic. So Multipoint LDP runs in parallel. In LSM with multipoint LDP, the same control and data plane is used as in the unicast MPLS core.  The overall application is called multicast VPN (MVPN). 

FRR can be used. 

### LSM: Packet Forwarding

For each packet coming in, MPLS creates multiple out-labels. If you were to look at a MDT in `show mpls forwarding table`, you could observe two outgoing labels over two different out interfaces. This indicates a point-to-multipoint LSP.

From the perspective of the source and receivers, things are unchanged. The PE routers talk PIM with each other then downstream to their CE's, then between the PE's is the provider core that is multipoint LDP aka Label Switch Multicast.

<br>

## PIM-SIM Overview

PIM is the most important protocol for multicast to function properly. The most common is `PIM-Sparse Mode (SM)`, and there is `PIM-Dense Mode (DM)` but mostly that is deprecated. PIM-SM with ASM is more optimal shared MDT.

Some terms:

- **Reverse Path Forwarding (RPF) Interface**: interface with lowest cost path (AD + metric) to IP address of the Source (SPT) or the RP (shared trees) 
- **RPF neighbor**: PIM neighbor on a RPF interface 
- **Upstream**: towards the Source of the tree; a PIM Join travels upstream 
- **Downstream**: towards Receivers 
-**Downstream Interface / Outgoing Interface (OIF)**: used to forward multicast traffic towards Receivers or down the tree from Source 
- **Incoming Interface (IIF)**: only interface that can accept multicast traffic from Source, which is the same as the RPF interface 
- **Outgoing Interface List (OIL**): a list of OIF's forwarding multicast traffic to the same group address 
- **Last-Hop Router (LHR)**: router directly attached to Receivers; responsible for PIM Join upstream 
- **First-Hop Router (FHR)**: router directly attached to Receivers; sending Register msg to RP 


PIM-SM is used in ASM.

- ASM requires a RP
    - RP can be statically configured everywhere, or there are dynamic methods with Auto-RP or PIMv2 bootstrap mechanism
    - RP helps sources find receivers by creating a MDT
- Initial flow is a shared tree down to receivers, then the LHR will optimize it by joining the source tree (SPT) directly to the source
    - Shared tree is a Shortest Path Tree (SPT) to the source, then down to the receivers is a shared tree.

## Packet flow for PIM-SM and ASM

### Receiver shared tree join

Receiver sends an IGMP membership join to (*,G). This is, send me any source to group G. 

LHR knows about the RP through config. It does a route lookup, then sends PIM join upstream to the RP. Most PIM messages are link local multicast 224.0.0.13 (`All-PIM`).

As the PIM join travels upstream from LHR to RP, routers create multicast state for (\*,G). The interface that the PIM join was received on becomes the OIL maintained by routers.

The PIM Join reaches the RP, and the shared tree is formed between receiver(s) and RP.

### Source Registration

Sources do not use a protocol to communicate with the network. They send multicast network, and the network does the rest.

When the FHR sees traffic to a multicast address, it checks if this is new or already existing. If there is no state, it sends a PIM-Register to the RP for group G (a mcast address).

Except, the source is already forwarding mcast traffic??!! so how can the network forward this traffic along a not pre-existing MDT?

Answer, high level, is that the FHR forwards the multicast traffic from the source encapsulated in unicast (!) PIM register messages to the RP. Once the RP receives such register messages, it decapsulates the packet and sends them down the shard tree to the receivers.

In parallel, the RP builds a (S,G) shortest path tree (SPT) back to the source. When done, there is a switchover to the SPT and unicast PIM Register tunnel is stopped.

### PIM-SM Switchover

LHR can decide to build a SPT back to source instead of using the shared tree via RP, `always` by default, or by configuration, it can be `threshold` or `never`. 

PIM Joins are sent hop by hop to the source, creating a new specific (S,G) SPT. 

For a period of time, multicast traffic will be duplicated down the RP SPT + RP->G shared tree and the new switchover SPT. The LHR will eventually receive the same multicast traffic from two interfaces, one on the RP-based shared tree, and one on the source-based SPT.

The switchover process to a SPT is triggered by a data bandwidth that exceeds the configured limit (or, if none is configured, immediately). The behavior can be configured per group. The LHR will set a `J` flag on (*,G) in the multicast routing table. If the threshold is set to `infinity`, then there will never be a switchover to join the SPT instead of shared tree.

To optimize, it now sends another PIM message along the shared tree, to prune (stop) receiving the specific (S,G) from the shared tree. Note that in ASM, there may be several sources, thus the prune must be specifically for the source. This is a specific (S,G) RP-bit Prune message to the RP. 

When the PIM Prune message reaches the RP, it checks if there are any other OIL for that group. If no, then RP will prune back to source.


<br>

## PIM-SM packets and state

The more optimal the traffic flows for multicast traffic, the more state on the network needs to be maintained.

| Packet Name | Destination | Description |
|-------------|-------------|-------------|
| PIM Hello | 224.0.0.13 (All-PIM-Routers) | Establishes PIM adjacencies between neighbors |
| PIM Join/Prune | Upstream PIM neighbor (unicast) | Joins or prunes multicast distribution trees |
| PIM Register | Rendezvous Point (unicast) | Encapsulates multicast data from source to RP |
| PIM Register-Stop | First-hop router (unicast) | Tells first-hop router to stop sending Register messages |
| PIM Assert | 224.0.0.13 (All-PIM-Routers) | Resolves forwarding conflicts on multi-access networks |
| PIM Bootstrap | 224.0.0.13 (All-PIM-Routers) | Distributes RP information in a PIM domain |
| PIM Candidate-RP-Advertisement | Bootstrap Router (unicast) | Informs BSR about candidate RPs |
| PIM State Refresh | 224.0.0.13 (All-PIM-Routers) | Maintains source state in PIM-DM domains (when enabled) |

### PIM neighbor discovery

PIM will establish neighbor adjacencies and elect a DR on each segment. The router with the highest IP address (or, if configured, the highest DR priority), is chosen as a DR. It is this router that will send PIM joins upstream. 

This is `PIM Hello` to 224.0.0.13. 

### PIM-SM forwarding

Multicast traffic forwarding is controlled by the OIL. Packets are sent out all interfaces on the OIL. 

There are three ways to put an interface on the OIL for a given multicast group:

1. When a PIM join for this group is received on this interface. The PIM join indicates that there are receivers further down that interface.
2. When an IGMP message has been received, indicating that a receiver joined on this segment.
3. There is also the possibility to statically join a group on an interface, via configuration. 

Expiration timers are reset depending on the type:

- (S,G) timers are reset when a multicast packet is forwarded. So as long as there is traffic, the state is maintained.
- (\*,G) timers are reset when a periodic PIM join is received for (\*,G). So as long as receivers are interested in this group, the state is maintained, independently of traffic sent. 

### PIM-SM Join process

PIM-SM Joins travel from receivers up to RP and/or the source.

In ASM, (*,G) creates state at each hop:

- IF there is no (\*,G) state, router processing the PIM Join will create it, add interface to OIL, send PIM Join to RP
- ELIF there is (\*,G) state, add interface to OIL. PIM Join process is complete, as there is ***already*** a built Shared tree
- Period (\*,G) PIM Joins refresh interfaces in the OIL between PIM neighbors

### PIM-SM state tables for all MDTs

The Multicast Routing Information Base (MRIB) is a protocol-independent multicast routing table that is created by PIM. It is composed of (S,G) or (*,G) entries with `Incoming interface`, `RPF neighbor`, then the `OIL`.

`show ip mroute` for IOS

`show mrib ipv4 route` on IOS-XR

So for an incoming multicast packet (possibly even the first one!!)...

1. if (S,G) exists for specific Source and Group, use this
2. if (*,G) exists for that Group, use this
3. if there is no entry, then the source must have just started sending and you are the FHR, congrats. The router (now the FHR) uses PIM Register packet to encapsulate and forward traffic to the RP.

<br>

## PIM-SM and Receivers

Receivers uses IGMP to communicate their Joins and to talk with the LHR. In ASM, IGMPv2 is sufficient, IGMPv3 is however backward compatible and can also be used. In ASM, a receiver joins a (*,G): any source sending to group G. When the LHR sees an IGMP message for a group G, it will add the interface on which this IGMP message was received to the OIL for the group.

## PIM-SIM and Source Registration

When a source starts to send, the network must register the source with the RP. THe FHR plays this role. It will encapsulate and forward source multicast traffic to the RP in PIM Register unicast packets.

There can be some weird scenarios, like say, there are 3 routers:

(`source`) - [`R1` *(FHR)*] - [`R2`] - [`R3` *RP*] - [Rest of provider network down to PE and CE] - (`receiver 1`)

Attached to R2 is a new ASM receiver that wants to join group G. There is already state for (S,G) on router B, so at first it seems that the only action needed on B is to add another interface to the OIL for the (S,G). However, the ASM group G may have more sources sending to the group. To be sure to find them all, the receiver must join the shared tree toward the RP! 

<br>

## PIM-SM Pruning in ASM

Pruning can happen explicitly, by sending PIM prune messages, or implicitly, using a time-out. This section explains how pruning works in an ASM environment. 

In ASM, if a receiver leaves a group, it may not trigger a Prune. When all receivers on a segment leave group G, then all interfaces are removed from OIL for entries (\*,G) and (S,G). Only once the last OIL is removed while the LHR send a PIM Prune message on the incoming interface for (\*,G).

Explicit prunes are only sent for (*,G) entries, which are only sent towards the RP (RP address noted in mroute table). (S,G) entries are left to time out.

<br>

## PIM-SM Implementation



``` py title="IOS XR"
hostname R1-XR
ipv4 access-list PIM_FILTER
  permit host 10.1.1.2
!
multicast-routing
  address-family ipv4
    interface all enable 
    interface Gi0/1
      enable
!
router pim address-family ipv4
  neighbor-filter PIM_FILTER
  interface Gi0/1
    enable
  rp-address 10.1.1.1
  spt-threshold [kbps | infinity]
```

``` py title="IOS XE"
hostname R2-XE
access-list 1 permit host 10.1.1.1
!
ip multicast-routing
ip pim neighbor-filter 1
ip pim spt-threshold [kbps | infinity]
ip pim rp-address 10.1.1.1
!
interface Gi0/0
  ip pim sparse-mode
!
```

### Show commands that are useful

***IOS-XE***
`show ip pim interface`
`show ip pim neighbor`
`mrinfo $ip_address`
`show ip pim rp`
`show ip rpf`
`show ip mroute`


***IOS-XR***
`show pim interface`
`show pim neighbor`
`mrinfo $ip_address`
`show pim topology`
`show pim rpf`
`show mrib route`

<br>

## Source-Specific Multicast (SSM)

When there are only one or a few sources sending multicast, ASM is probably too complex and unnecessary. SSM does not require a RP, and there is a single MDT for each source. No shared (\*,G) trees are built or used. Since are no RP's, for inter-domain routing, there is no need for Multicast Source Discovery Protocol (MSDP). Both ASM and SSM can coexist on the network though.

232.0.0.0/8, FF3x::/32 are used for SSM groups.

SSM is an enhancement to PIM, written as PIM-SSM usually. PIM-SSM is made possible by IGMPv3 and MLDv2.

The network service identified by (S,G), for SSM address G and source host address S, is referred to as a "channel".

A high level SSM flow:

1. A receiver learns about (S1,G) and (S2,G) from an out-of-band method, like a web page. 
    - (S1,G) and (S2,G) are different channels, so they would be different trees.
2. The receiver sends IGMPv3 "membership report" requesting to join these groups specifically. 
3. LHR receives the IGMPv3 "membership report", then creates state for the requested channels. It adds the interface the IGMPv3 packet was received on to the OIL.
4. The LHR creates a PIM (S,G) join message upstream. The upstream router is determined by the source IP address, which is a unicast table lookup to the source at each router in the path.
5. The result will be a shortest path tree (SPT) to the FHR next to the source. 

The network keeps state for every (S,G) channel independently. So, for a multicast group with many sources, each source within that group would end up with its own SPF tree in SSM. Therefore, SSM is not optimal for multicast distributions where many sources send to the same group. 


### IGMPv3 

IGMP has no transport (L4), like ICMP. It uses IP protocol 2

| Packet Type | Message Name | Description |
|-------------|--------------|-------------|
| 0x11 | Membership Query | Sent by routers to query group membership |
| 0x16 | v2 Membership Report | Sent by hosts to join group also called `IGMP Join` |
| 0x17 | v2 leave group | Sent by hosts to stop mcast traffic from a group they joined |
| 0x22 | Version 3 Membership Report | Sent by hosts to report group membership |
| 0x30 | Multicast Router Advertisement | Sent by routers to advertise their presence |
| 0x31 | Multicast Router Solicitation | Sent by hosts to solicit router advertisements |
| 0x32 | Multicast Router Termination | Sent by routers to indicate they are leaving the network |


### SSM Mapping

If not all end hosts support IGMPv3, then those devices cannot subscribe to SSM channels because they cannot request (S,G). Legacy group membership reports for groups in the SSM group range are mapped to a set of sources providing service for that set of (S,G) channels. 

Two ways to configure:

1. Static
2. DNS-based


``` py title="IOS XR"
Router#configure
Router(config)#ipv4 access-list 4
Router(config-ipv4-acl)#permit ipv4 any 232.1.2.0 0.0.0.255
Router(config-ipv4-acl)#exit
Router(config)# multicast-routing
Router(config-mcast)#address-family ipv4
Router(config-mcast-default-ipv4)#ssm range 4
Router(config-mcast-default-ipv4)#exit
Router(config-mcast)#exit
Router(config)#router igmp
Router(config-igmp)#ssm map static 172.16.8.10 4
*/Repeat the above step as many times as you have source addresses to include in the set for SSM mapping/*
Router(config-igmp)#int te0/0/0/3
Router(config-igmp-default-if)#static-group 232.1.2.10
Router(config-igmp-default-if)#commit

# DNS-based
Router#config
Router(config)#domain multicast cisco.com
Router(config-igmp)#domain name-server 10.10.10.1
Router(config-igmp)#router igmp
Router(config-igmp)#ssm map query dns

```

``` py title="IOS XE"
Router# conf t
Router(config)# access-list 10 permit 232.1.2.10
Router(config)# ip igmp static ssm-map group-list ssm-source 10 172.16.8.10
Router(config)# ip igmp ssm-map enable 


# DNS-based
Router(config)# ip domain multicast ssm-map.cisco.com 
Router(config)# ip name-server 10.48.81.21 # generic DNS server config
```

In examples above 172.16.8.10 = $source and 232.1.2.10 = SSM group

For DNS-based dynamic mapping, a LHR takes the DNS-name from group address in the IGMPv2 Membership Report. The router looks up IP address resource records (IP A RRs) to be returned for this constructed domain name and uses the returned IP addresses as the source addresses associated with this group.

### Configuring SSM

Most of the SSM config is shown above, but to summarize a generic PIM-SSM configuration, it will look like:

IOS-XE: 

`ip multicast-routing`
`ip pim ssm default`

IOS-XR:
`multicast-routing`
`address-family`
`ssm (allow-override) (range) (disable)`
`interface $if`
  `enable`


<br>

## Bidirectional PIM

*Hard to find good sources for material but https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipmulti_pim/configuration/xe-3s/asr903/17-1-1/b-imc-pim-xe-17-1-asr900/m-bidirectional-pim.pdf looks good*

For applications with many endpoints that both send and receive, BIDIR-PIM was developed. In BIDIR-PIM, a host can be both a source and a receiver, or just one of the two. If a receiver starts also sourcing traffic, no additional state is required in the network. 

PIM Bidirectional (BIDIR) has one shared tree from sources to RP and from RP to receivers. This is unlike the PIM-SM, which is unidirectional by nature with multiple source trees - one per (S,G) or a shared tree from receiver to RP and multiple SG trees from RP to sources. 

PIM-SM loops are avoided with RPF check. BIDIR maintains loop-free topology with a Designated Forwarder (DF).

The thing with many-to-many is that a not all sources and receivers can be both sending and receiving at the same time. The bidirectional "sources" can change, which is why the example of a video conference application is used. When there is a new speaker, there is a new sender/receiver. The "bidirectional" multicast flow will shift on the network, but the "shared tree" more or less stays the same.

- When a new source starts sending for a BIDIR-compatible application, then there will a new FHR, which will recreate the shared tree to/from the RP.

### Designated Forwarder (DF)

Loop avoidance in BIDIR is via the DF. On each segment, a DF is elected, whom is responsible for forwarding multicast traffic in both directions to ***and*** from the RP.

The algorithm to elect the designated forwarder is straightforward, all the PIM neighbors in a subnet advertise their unicast route to the rendezvous point and the router with the best route is elected. If there is a tie, highest IP address wins.

This effectively builds a shortest path between every subnet and the rendezvous point without consuming any multicast routing state (no (S,G) entries are generated). The designated forwarder election mechanism expects all of the PIM neighbors to be BIDIR enabled

### BIDIR Forwarding - Source

When a source starts to send, BIDIR will forward multicast to the RP. Routers must known their RP, either through manual config or dynamic methods.

Each router will create (\*,G) multicast forwarding state. Except with BIDIR, there is no IIF (incoming interfaces), there is just `Bidir upstream`, always pointing to the RP. There is OIL, but it refers to the interface on which traffic from `$source` is received added to OIL (which is the DF).

BIDIR works very with many-to-many applications, since there is only one (*,G) entry for each BIDIR group, which is the reason why BiDir is preferable with many sources: It creates significantly less state than PIM-SM or PIM-SSM. 

### BIDIR Forwarding - Receiver

When a router receives IGMP Join for bidirectional group G, LHR determines if it is the DF. The multicast state `(*,G)` will be flagged for BIDIR. If there is not one, then it will check DF election state, which will inform it which router on the segment is DF.  

When a router receives a Join or Leave message, and the router is not the DF for the receiving interface, the
message is ignored. Otherwise, the router updates the shared tree in the same way as in sparse mode.

At each hop, the interface that the join was received on is added to the OIL, like in normal PIM-SM operations. The result is an RP-based shared MDT, which is loop free, because only the DFs forward the joins (and later the traffic).

This goes hop by hop effectively the same as PIM-SM up to the RP, which creates a shared tree that is loop-free.

### Configuring PIM-BIDIR


``` py title="IOS XE"
!
ip pim bidir-enable
ip pim rp-address 20.1.1.10 BIDIR-GROUPS bidir
!
ip access-list standard BIDIR-GROUPS
20 permit 225.0.0.0 0.255.255.255
!
interface GigabitEthernet0/2/1
  ip pim sparse-mode
interface GigabitEthernet0/2/4
  ip pim sparse-mode
!
```

`show ip mroute` will show `Bidir-Upstream:`

`show ip pim neigh` will show `B` flag that indicates Bidir
`show ip pim int df` shows DF 


- IOS-XR, supposedly PIM-BIDIR is enabled by default but... I don't see evidence of this. The Cat8000 router IOS-XR docs says, PIM-BIDIR is not supported! Just configure the RP to be Bidir compat.

``` py title="IOS XR"
router pim address-family ipv4
  rp-address 20.1.1.10 bidir
```

## More on DF

A DF is elected on each link for a specific RP. If there are more than 1 RP, there will be RP-specific DF's on each link. To optimise tree creation, it is desirable that the winner of the election process should be the router on the link with the "best" unicast routing metric (as reported by the MRIB) to reach the RP.

Election updates usually occur only once, but they can be triggered if there are changes:

- Unicast metric change to reach the RP
- If the DF interface also becomes the interface to reach the RP
- A new PIM neighbor establishes on the segment
- Elected DF fails

### DF Election Messages

**Offer** `(OfferingID, Metric)`

- Sent by routers that believe they have a better metric to the RPA than the metric that has been on offer so far.

**Winner** `(DF-ID, DF-Metric)`

- Sent by a router when assuming the role of the DF or when re-asserting in response to worse offers.

**Backoff** `(DF-ID, DF-Metric, OfferingID, OfferMetric, BackoffInterval)`

- Used by the DF to acknowledge better offers.  It instructs other routers with equal or worse offers to wait until the DF passes responsibility to the sender of the offer.

**Pass** `(Old-DF-ID, Old-DF-Metric, New-DF-ID, New-DF-Metric)`

- Used by the old DF to pass forwarding responsibility to a router that has previously made an offer.  The Old-DF-Metric is the current metric of the DF at the time the pass is sent.


During initial election, the backoff interval is 3 times the offer interval. 

Different routers on a segment advertise their metric. If they receive an Offer with better metric, it will stop sending Offers (a backoff interval x3 Offer interval). 

If a "winning" router sends 3 uncontested Offers, it will then send a `Winner` packet with ID and metric to RP.

If DF loses route to RP, it sends a new Offer with infinite metric, which triggers a re-election.

If DF "fails", the routing change will trigger downstream routers on segment to re-elect based on metric change for route to reach RP.

<br>

## Interdomain Multicast

Keep things simple. SSM for interdomain. Re-use existing protocols-- MP-BGP to carry multicast info.

If ASM is required, RP's have to be involved. This requires Multicast Source Discovery Protocol (MSDP) to exchange source info between domains via RP's.

### SSM for interdomain

Not actually required to use MP-BGP to transport multicast NLRI's for a seperate multicast table. SSM and the RPF check can use the unicast routing table. 

On the AS border routers, `ip multicast-routing` must be configured, PIM SSM must be configured globally, and PIM-SM on every interface.


``` py title="IOS XR"
router bgp 65000
  address-family ipv4 unicast
  address-family ipv4 multicast
!
  neighbor 192.0.2.1
    remote-as 65001
    address-family ipv4 unicast
  neighbor 192.0.2.5
    remote-as 65001
    address-family ipv4 multicast
!
```

``` py title="IOS XE"
router bgp 65001
  neighbor 192.0.2.2 remote-as 65000
  neighbor 192.0.2.6 remote-as 65000
  address-family ipv4 unicast
    neighbor 192.0.2.2 activate
    no neighbor 192.0.2.5 activate
!
  address-family ipv4 multicast
    neighbor 192.0.2.5 activate
!
```


`show bgp ipv4 multicast` to look at IPv4 multicast BGP RIB.

```
ip multicast-routing
!
interface Gi0/1
  description Backbone-other-Routers
  ip pim sparse-mode
!
interface Gi0/2
  description Host-facing-LAN
  ip pim sparse-mode
  ip igmp version 3
!
ip pim ssm default
```

### ASM for interdomain

**MSDP is IPv4 only.**

1. Source starts sending. FHR sends PIM Register to local AS1 RP.
2. Local AS1 RP informs MSDP peers about new source G-1.
3. AS2 receiver joins (*,G-1) via IGMP
4. PIM join travels to AS2 RP
5. AS2 RP has learned about G-1 source from AS1 RP via MSDP, connects receiver to source G-1
6. MDT gets built interdomain

``` py title="IOS XE"
ip access-list extended MSDP_ACL
  accept ip host 239.1.1.2
  accept ip host 239.1.1.1
!
ip msdp peer 192.0.2.1 connect-source Loopback0
ip msdp originator-id Loopback0
ip msdp ttl-threshold 192.0.2.1 140
ip msdp sa-filter in 192.0.2.1 list MSDP_ACL
ip msdp sa-filter out 192.0.2.1 list MSDP_ACL
ip msdp password peer 192.0.2.1 Cisco123
```

https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/24xx/multicast/configuration/guide/b-multicast-cg-asr9k-24xx/Implementing-layer3-multicast.html#task_2948296

``` py title="IOS XR"
ipv4 access-list 100 20 permit 239.1.1.1 0.0.0.0
ipv4 access-list 100 30 permit 239.1.1.2 0.0.0.0
!
router msdp
  originator-id Lo0
  peer 192.0.2.2 
    connect-source Lo0
    ttl-threshold 8
    password Cisco123
    remote-as 65001
  sa-filter in list 100
  sa-filter out list 100
```

`show msdp summary` to look at peers.

`show msdp sa-cache` to see what multicast state is being exchange.

`show msdp rpf` to view RPF lookup associated.

## Multicast architecture

ASM deployments depend on RP placement and function, so ensuring HA is critical. 

Dynamic RP configurations are many-- `Auto-RP`, `BSR`, `Anycast RP`, but there are often drawbacks and restrictions.

There is also Multicast NSF with stateful switchover (SSO).

### Auto-RP

Cisco-only. It automates the distribution of group-to-RP mappings in a PIM network, or said another way, it allows routers to find RPs automatically for generic or specific groups. Highest IP address for any group via RP's will win and be chosen. Actually... that is kind of shit for design.

This simplifies the configuration, and it can provide HA and other optimal design. It allows for load distribution, plus facilitates the arrangement of RPs according to the location of group participants. 

Configure routers as ***candidate RPs*** so that they can announce their interest (via `rp-announce` on 224.0.1.39) in operating as an RP for certain group ranges. 

Additionally, a router must be designated as an ***RP-mapping agent*** that receives the `rp-announce` messages from the candidate RPs, and arbitrates conflicts. The RP-mapping agent sends the consistent group-to-RP mappings to all remaining routers via `rp-discovery` on 224.0.1.40. Thus, all routers automatically determine which RP to use for the groups they support. 

A router can be configured to be both Candidate RP and mapping agents at the same time. 

`rp-announce` message contains:

- group IP address range, default is 224/4
- IP address of candidate RP
- a holdtime used to detect failures, which is 3x the announcement interval (180s in total)

### Configuring Auto-RP

For IOS-XE, there is a "strange" config requirement. Auto-RP requires PIM-DM to work. 

You could enable PIM-DM, but that's not a good idea, given how it works. So IOS-XE uses `ip pim autorp listener` and `ip pim autorp listener` to ensure Auto-RP 224.0.1.39 `rp-announce` and 224.0.1.40 `rp-discovery` are forwarded.

IOS-XR does not support PIM-DM, but it has builtin exceptions to forward these.


``` py title="IOS XR"
!
router pim address-family ipv4
  auto-rp candidate-rp GigabitEthernet0/1/0/1 scope 31 group-list 2 bidir
  auto-rp mapping-agent GigabitEthernet0/1/0/1 scope 20
!
ipv4 access-list 2
  permit 239.1.1.1 0.0.0.0
```

`scope` = TTL hops

``` py title="IOS XE"
!
# the below is required for IOS-XE to enable forwarding of 224.0.1.39 and 224.0.1.40 in dense mode on all multicast enabled interfaces. 
ip pim autorp listener
!
ip pim send-rp-announce Gi0/1 scope 31 group-list 10
ip pim send-rp-discovery Gi0/1 scope 20 group-list 20
!
ip access-list standard 10
  permit 239.1.1.1
!
ip access-list standard 20
  permit 192.0.2.120
  permit 192.0.2.240
```

`show ip pim rp mapping` on mapping-agent to look at specific RP's and their group announcements.

### Some best practice notes

Like unicast, you should secure your multicast boundary, this includes control plane multicast traffic like Auto-RP. Explicitly filter and control which groups are allowed to cross domains and deny everything else.

If there are RP's outside your control that you want to enable Auto-RP for some specific groups to work for, you can configure this...

```
interface GigabitEthernet0/0
 ip multicast boundary AutoRP-ACL-filter filter-autorp
!
access-list AutoRP-ACL-filter permit 224.1.1.1
```

### PIMv2 Bootstrap Router

RFC 5039; fault tolerant, automated RP discovery and distribution mechanism. It requires PIMv2, which BSR uses for BSR signalling, while Auto-RP can work with v1/v2.

A single router is elected as the BSR from a collection of candidate BSRs. If the current BSR fails, a new election is triggered. The election mechanism is preemptive based on the priority of the candidate BSR.

In order to determine the RP for a multicast group, a PIM router maintains a collection of group-to-RP mappings, called the `RP-Set`. A group-to-RP mapping contains the following elements.

- Multicast group range, expressed as an address and prefix length
- RP priority
- RP address
- Hash mask length
- SM / BIDIR flag

All configured BSR routers send BSR messages on PIMv2 224.0.0.13, which has a TTL 1. BSR info is propagated via normal PIM methods. 

**First**, BSR routers send BSR messages throughout the PIM domain with the address of the BSR candidate sending the message and its priority.

**Second**, all candidate BSR's send their `bsr-messages` with the `candidate` label. Similiar to other election protocols, candidates announce themselves until they hear a higher priority. 

- If the priority of another candidate BSR is higher, it backs off.
- If the priority of another candidate is lower, it elects itself as the elected BSR.
- If the priority is the same, it decides on the IP address (like in Auto-RP)—higher address wins. 

Next, **third**, RP candidates watch the BSR election waiting for a winner, an `elected BSR`. When candidate RPs see BSR messages from an elected BSR, they respond unicast to this BSR, with their role as candidate RP all group information they are configured with. 

Finally, **fourth**, the elected BSR sends the resulting RP to group mapping out to the rest of the multicast domain, again using the same PIMv2 BSR messages.

Other routers utilize the RP-set info

``` py title="IOS XE"
!
# for RP-candidates
ip pim rp-candidate $interface
!
# for BSR candidates
ip pim bsr-candidate $interface
!
interface Gi0/1
  description Interdomain border
  # stops bsr packets on inter-domain border
  ip pim bsr-border
```

``` py title="IOS XR"
!
router pim address-family ipv4
  # for RP-candidates
  bsr candidate-rp $ip_address priority 200
  !
  # for BSR candidates
  bsr candidate-bsr $ip_address priority 100
  interface Gi0/0/0/1
    # stops bsr packets on inter-domain border
    bsr-border
  !
!
```

There is no specific configuration required on all other routers, except enabling of multicast globally and on the respective interfaces. Since BSR messages are normal PIM messages, a router will interpret them on any multicast enabled interface.

`show pim bsr election` -- list of candidate BSRs
`show pim bsr rp-cache` -- all available RPs

<br>

### Anycast RP

RP's can be anycasted with Multicast Source Discovery Protocol (MSDP) to provide RP redundancy, rapid RP failover, and RP load balancing. This concept is defined in RFC 3446.

MSDP maintains sources state between anycasted RP's. Therefore, MSDP is required to do source active announcements between RPs. This way both RPs know mutually the sources on the other RP and can construct a joint MDT. 


``` py title="IOS XE"
!
interface Loopback0
  ip add 10.0.0.1 255.255.255.255
!
interface Loopback1
  ip add 10.1.1.1 255.255.255.255
!
ip msdp peer 10.0.0.2 connect-source Loopback0
ip msdp originator-id Loopback0
ip pim rp-address 10.1.1.1
```

``` py title="IOS XR"
interface Loopback0
  ip add 10.0.0.2 255.255.255.255
interface Loopback1
  ip add 10.1.1.1 255.255.255.255
!
router msdp
  peer 10.0.0.1 connect-source Loopback0
  originator-id Loopback0
router pim address-family ipv4
  rp-address 10.1.1.1
!
```