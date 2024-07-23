# MPLS TE

## TE Labels and Forwarding Concepts

MPLS TE uses a different set of protocols, not LDP, to enable source-based selection of the entire LSP used for TE. 

A link-state IGP must be used to determine the shortest path, then the output is used with RSVP to establish the LSP.

A process of establishing a Cisco MPLS TE tunnel and forwarding traffic into it has a few steps:

- **Information distribution**: resource attributes are configured locally, then they are distributed to the headend routers of traffic tunnels. The resource attributes are flooded throughout the network using extensions (TLV's) on existing IGPs OSPF and IS-IS. 
    - The flood of resource info occurs when: (1) link-state changes, (2) resource of a link changes (due to admin intervention), (3) amount of bw crosses a preconfigured threshold
- **Path selection**: uses Constrained Shortest Path First (CSPF) to get an explicit route consistenting of a sequence of label switching routers. CSPF ignores links explicitly excluded by resource class affinities of tunnel traffic or links with insufficient bw.
- **Tunnel admission control**: manages the situation when a router along a computed path has insufficient bandwidth to honor the resource that is requested in the RSVP PATH message
- **Forwarding the traffic into tunnel**: traffic be forwarded a few different ways-- static routes, policy routing from global routing table, autoroute
- **Path maintenance**: There are two operations here-- path reoptimization and restoration

### Resource Reservation Protocol (RSVP) in Path Setup

A signalling protocol is used to confirm the path, to check and apply bw reservations, and to exchange MPLS labels to build LSP's.

RSVP sets up the TE LSP with:

1. PATH message (from head to tail) carrying LABEL_REQUEST
2. RESV message (from tail to head) carrying the LABEL

The result is a path with IP addresses of next-hop IP addresses along the path. 

However, this path is only known to the headend router. The intermediate routers do not make their own CSPF calculations; they merely abide by the path that is provided to them by the headend router. 

### Path Setup, Admission Control w/RSVP

The RSVP PATH message contains the explicit route calculated by CSPF on headend route. Each intermediate router performs admission control after receiving the PATH message. 

Tailend router receives PATH msg, then sends RESV msg back towards headend router. Each intermediate router along the path reserves bw and allocates labels for tunnel traffic. 

RSVP messages also provide support for LSP teardown and error signaling.

`RSVP Error` for the unavailability of the requested resources. An intermediate router may send `PATHERR` during admission control if it cannot accomodate resources requested in PATH msg from headend to tailend router. An intermediate router may also send `RESVERR` in response to lack of resources to tailend router in response to a RESV msg.

`RSVP Tear` has two types-- PATH tear and RESERVATION tear messages. 

- The process of clearing a PATH or RESERVATION state on a router using tear messages enables the reuse of resources on the router for other requests. The PATH tear messages are usually generated in inter-area LSP creation where the inter-area LSP is not configured to be fast reroutable, and if a link failure occurs within an area, the LSR to which the failed link is directly attached will generate an RSVP PATH error and an RESV tear message to the headend. The headend will then generate an RSVP PATH tear message. 

On intermediate routers, trunk admission control is used to ensure each device has sufficient bw to support resources requested in PATH msg. There are priority levels 0-7. If there is enough bw, the reservation is accepted, otherwise the path setup fails. Then when the RSVP RESV msg comes back, the router reserves the bw for the LSP.

As part of admission control, the router does local accounting to track resources and may trigger IS-IS or OSPF updates when available resources cross a configured threshold.

### Forwarding Traffic to a Tunnel

The LSP or traffic tunnel will not appear in the IP routing table, since it is not known to routing processes. 

IP traffic can be mapped onto a traffic tunnel in four ways:

- Use static routes that point to the tunnel interfaces.
- Using policy-based routing (PBR), set the next hop for the destination to the tunnel interface.
- Use the autoroute feature, an SPF enhancement that includes the tunnel interface. The result of the autoroute feature is that the tunnel is seen at the headend (and only there) as a connected interface. The metric of the tunnel is set to the normal IGP metric from the tunnel headend to the tunnel endpoint (over the least-cost path, regardless of whether the tunnel is actually using the least-cost path). With the autoroute feature, the traffic-engineered tunnel appears in the IP routing table, but this appearance is restricted to the tunnel headend only.
    - Enables all prefixes topologically "behind" the tailend router to be reachable via the autoroute tunnel on headend.

**Autoroute with Forwarding Adjacency**

- Allows the tunnel to be announced via OSPF or IS-IS as a p2p link to other routers. The tunnel *MUST* be setup bidirectionally, since it is possible with MPLS RSVP for the LSP to take different unidirectional paths.
- TE tunnel interfaces are advertised like any other link into the IGP. 


## MPLS TE Attributes

Traditional link-state IGP considers nodes and links to calculate shortest path. BUT, the CSPF algorithm considers additional information such as available resources or constraints. Then, the IGP is responsible for distributing available resource and constraint info. 

Links have their own attributes, then tunnels will have their own set of attributes. Tunnel attributes seemingly must fit within the attributes defined per link end-to-end.

**Link-resource attributes**:

1. Maximum bw
2. Maximum reservable bw
3. Link-resource class
4. Constraint-based specific link metric

**Tunnel attributes**:

1. traffic parameter
2. general path selection and management
3. tunnel resource 
4. adaptability
5. priority
6. pre-emption
7. resilience

### Link-resource attributes

There are 3 **Bandwidth** link-resource attributes:

- `Maximum bandwidth`
- `Maximum reservable bandwidth`
- `Unreserved bandwidth`

There could be non-MPLS traffic on the links, so leaving room could be useful. The attribute is a measure of allocation, not the actual utilization.

Also, there are priority levels for traffic tunnels, this availability information needs to be configured for each priority level on the link. The bw at the upper priority level is typically higher than at lower levels (0-7 levels). Because of oversubscription, the total amount of bandwidth can exceed the actual bandwidth of the link.

The `Link-resource class` attribute is a 32 byte value that is the traffic tunnel resource class affinity attribute, and allows inclusion or exclusion of the link into the path of the tunnel.

`Administrative weight` is a constraint-based metric that is not related to IGP cost/metric. This will default to IGP link cost if not defined. 

### TE Tunnel Attributes

`traffic parameter` (bandwidth) attributes specifies the amount of bw required by the TE tunnel. There could be other traffic characteristics, such as peak rates, avg rates, permissible burst rate, etc. 

- These characteristics are useful for resource allocation. A path is not considered for a Cisco MPLS TE tunnel if it does not have the bandwidth that is required.

`path selection and management` attributes (path selection policy) specifies the way in which the headend routers should select explicit paths for traffic tunnels.

`tunnel resource class affinity` attribute allows for an operator to apply path-selection policies by administratively including or excluding links.

- Each link may include `link-resource class` attribute, then a tunnel `resource class affinity` could explicitly include or exclude links when doing path-selection. Basically, this sounds like manual traffic steering to ensure TE doesn't go off the rails and pick a really bad path or something similiar.

`Adaptivity` attribute indicates whether the TE tunnel is subject to re-optimization or re-routed through different paths by the underlying IGP based on changing resource availability.

`Priority` attribute defines the order in which path selection during establishment and under faulty scenarios. This is very important to define if there is preemption during RSVP process to establish TE tunnels.

`Preemption` attribute determines whether another traffic tunnel can pre-empt a specific TE tunnel. 

There are two types of priorities to define for TE tunnels:

1. Setup priority (`priority`) defines relative important of tunnels and determines the order in which path selection is done for traffic tunnels at connection establishment and during rerouting because of faulty conditions.
2. Holding priority (`preemption`) defines "pre-emptive" rights of competing tunnels and specifies priority of a router to hold a tunnel resource. Useful when trying to differentiate between services and following faults under re-route or contention events. 

`resilience` attribute defines the behavior of a TE tunnel in faulty conditions or if the tunnel becomes noncompliant with tunnel attributes (like, required bw).

Some examples of what the `resilience` attribute may do

- Do not reroute the TE tunnel
    - For example, a survivability scheme may already be in place, provisioned through an alternate mechanism, which guarantees service continuity under failure scenarios without the need to reroute tunnels.
- Reroute through a feasible path with enough resources. If none exists, then do not reroute.
- Reroute through any available path regardless of resource constraints.

### Implementing TE policies with Affinity Bits

Each tunnel can have a 32-bit resource class `affinity` string, which has a mask to exlcude the respective link resource class bits from being checked. 

Resource class affinity attributes associated with a TE tunnel can be used to specify the class of resources (see Section 6 of rfc2702) which are to be explicitly included or excluded from the path of the TE tunnel. These are policy attributes which can be used to impose additional constraints on the path traversed by a given TE tunnel.

The link is characterized by following:
- The link resource class (default value is 0).
- The resource class is propagated using a link-state IGP

The tunnel is characterized by the following:
- Tunnel resource class affinity (default value is 0)
- Tunnel resource class affinity mask
    - Mask meaning: 0 = do not care, 1 = care (link's resource class bit must match the tunnel bit)
    - Default value of the tunnel mask is 0x0000FFFF

So for example, tunnel affinity = `Bits: 0000 Mask: 0011`. If there is a link resource class with `0010`, then the tunnel cannot traverse that link in the path. 

Resource class affinity attributes are very useful and powerful constructs because they can be used to implement a variety of policies. For example, they can be used to contain certain traffic trunks within specific topological regions of the network.

## MPLS TE Path Setup, Computation, and Optimization

Once a headend router has the necessary info (topology, link costs, resources and constraints), then it can use this to calculate best paths for configured TE tunnels.

IGP's must distribute all link characterists to other routers in the MPLS TE domain.

- Resource flooding is triggered whenever there is significant change that could affect other routers when they perform CSPF.
- Resource flooding triggers are:
    - link state change
    - config changes: cost, resource class, bw
    - amount of available bw crosses a preconfigured thershold
    - periodic (timer-based)... a node checks attributes, if changes, it floods updates
    - LSP setup failure when processing RSVP RESV

OSPF uses opaque LSAs and IS-IS uses new TLV's

Another important factor in LSP computation is the available bandwidth on the link over which the traffic tunnel will pass. This bandwidth is configured per priority level (8 levels, 0 being the highest, 7 the lowest) and communicated in respective IGP link-state updates, again per priority.

When a certain amount of the bandwidth is reserved at a certain priority level, this amount is subtracted from the available bandwidth at that level and at all levels below. The bandwidth at upper levels remains unchanged.

For stability, significant rapid changes in available link resources should not trigger the updates immediately. There are timers for this.

The drawback could be that a headend router seeing a link as available for a LSP and includes the link in path computation, even if the link is down or does not have enough resources. When the RSVP PATH message is sent, the node with the cannot establish the link and immediately floods an update.

### Constraint-based path computation

So, now the headend router will have a full view of the topology and network resources which is flooded throughout the network via the IGP.

An explicit route expressed as a sequence of interface IP addresses (for numbered links) or TE router ID's (for unnumbered links) in the path from headend tunnel endpoints.

Two methods for establishing tunnel traffic: (1) static, (2) dynamic path setup

RSVP is used for LSP signalling to tailend router to establish the final path.

LSP computation is limited by several factors (constraint-based):

1. Endpoints are in the same IS-IS or OSPF area due to link-state topology limitations
2. The links that are explicitly excluded via the link resource class bit string, or that cannot provide the required bandwidth, are pruned from the computation.

### Path Selection w/Constraint-Based computation

Prior to path selection, link evaluation will automatically exclude links that are excluded via resource class affinities. 

Establishment of a tunnel does not trigger any LSA announcements or a new SPF calculation (unless the forwarding adjacency feature is enabled).

The word tunnel is a little misleading, in that it is more like a visualization of a concept. 

With the autoroute feature, the TE tunnel on the headend router has the following characteristics:

- Appears in the routing table
- Has an associated IP metric (cost equal to the best IGP metric to the tunnel endpoint)
- Is also used to forward the traffic for destinations behind the tailend tunnel endpoint


Path-selection for Constrained-based Routing(CBR):

1. It's own metric first, which is `administrative weight`
2. If tie, then select the path with highest minimum bandwidth
3. If tie, select path with the smallest hop count
4. If above fails, then pick a random path.

**RSVP in the Path Setup**

After an explicit route is selected via CBR, the explicit route is used with RSVP to assign labels and reserve bw on each link. Additionally, RSVP transports traffic parameters and maintains the control and policy over the path. The maintenance is done by periodic refresh messages that are sent along the path to maintain the state.

RSVP then signals routers in the path from headend to tailend to reserve resources to establish the **LSP path** and allocate labels. If good, a unidirectional TE tunnel is established, as only seen on headend router.

### Hop-by-Hop Path Setup with RSVP

[R1]Eth1 - Eth2[R2]Eth1 - Eth2[R3]

R1 headend sends RSVP PATH to R3 tailend.

There are RSVP objects in the PATH message:

- Session(tailend, headend) like `(R3-lo0, R1-Lo0)`
- Explicit route object (ERO), which is a explicit list of next hops chosen by CBR. 
    - Like `(R2-Eth2, R3-Eth2)`
- PHOP (Previous Hop), like (`R1-Eth1`)
- Other fields...

As the next-hop router (R2) receives the RSVP PATH message, the router checks the ERO and looks into the L bit regarding the next-hop information. If this bit is set and the next hop is not on a directly connected network, the node performs a CBR calculation (path calculation, or PCALC) using its TE database and specifies this loose next hop as the destination.

In this way, the ERO is augmented by the new results and forms a hop-by-hop path up to the next loose node specification.

**Intermediate routers along the path** (indicated in the ERO) perform the traffic tunnel admission control by inspecting the contents of the session attribute object. If the node cannot meet the requirements, it generates a PATH_ERR message. If the requirements are met, the node is saved in the Record Route Object of the RSVP PATH message.

At the tailend router, upon receiving the PATH message, the `label_request` triggers path label allocation. The label is placed in the corresponding label object of the RSVP RESV message that is generated. The RSVP message is sent back to the headend following the reverse path that is recorded in the RRO, and is stored at each hop in its path state block.

Each hop allocates a label, which is stored and replaced in the `Label` field of the RSVP RESV message. Also, as the RESV packet traverses the reverse path, the `NHOP` field is moved into the `Record Route` (RRO) field of the RESV packet. Upon arrival back at the headend, the RRO in the RESV packet should match the Explicit Route in the RSVP Path message initially sent.


### TE tunnel and link admission control

When the RSVP PATH message is sent from headend router to tailend router, each hop on the way determines if the available resources specified in the `Session` attribute object are available. 

If there is not enough bandwidth, the link-level (per-hop) call admission control (LCAC) module informs RSVP about the lack of resources, and RSVP generates an `RSVP PATH_ERR` message with the code Requested bandwidth unavailable. Additionally, the flooding of the node resource information (by the respective link-state IGP) can be triggered.

If there is enough bw, then the bw is reserved and put into the waiting pool for the RESV message to come back around. If the resource threshold is reached, then the IGP triggers the flooding of resource info. 

During admission control, priorities are checked. If there is a prempt action to be taken, then lower priority sessions are booted with reservation triggers a `RESV_ERR` or a `PATH_ERR` message or both with the code Policy control failure.

### Path Rerouting and Reoptimization

`reoptimization` is done on a periodic basis if used. At certain intervals, a check for most optimal paths for LSPs tunnels is done, then if the current path is not the most optimal, then tunnel rerouting is initiated. 
    - Default is 1 hour for `reoptimization`

After the new LSP is successfully established, the traffic is rerouted to the new path and the reserved resources of the previous path are released. The release is done by the tail-end router, which initiates an `RSVP PATH_TEAR` message.

When a link failure occurs, a new path calc is done, then a new LSP is created. This can be detected one of two ways:

1. IGP sends new link-state packet with about path changes
2. RSVP alarms the failure via `RSVP PATH_TEAR` to the headend

But overall, both indicate that the LSP is no longer available, and likely reasons could be:

- link or router in path is down
- LSP is preempted by another LSP

RSVP session is torn down, and there is a new path calculation. If a new path is found, a new LSP is signaled via RSVP, and the headend router updates adjacency table is updated to this new tunnel, then CEF table updated for all routes using this tunnel adjacency. 

***NOTE***, during a reroute event where the old LSP is not available, traffic intended for the tunnel will be using the old LSP, which will blackhole the traffic.

The process of freeing resources in the established LSP paths is called MPLS-TE preemption. In order to reduce traffic interruption, `soft pre-emption` feature is introduced to the RSVP-TE protocol to minimize traffic disrupted on preempted LSP.

The `soft preemption` feature tries to pre-empt established LSP in a graceful manner to avoid or minimize traffic loss. In some cases, the link can be oversubscribed for a longer period of time.

### Assigning traffic to TE tunnels

Aka, routing traffic into tunnels

From an IP perspective, the LSP is just a tunnel. 

You can use static routes pointing to the tunnel interface. Like not scalable or nice for troubleshooting.

The autoroute feature enables all the prefixes that are topologically behind the Cisco MPLS TE tunnel endpoint (tail end) to be reachable via the tunnel itself (unlike with static routing, where only statically configured destinations are reachable via the tunnel). 

The cost of the tunnel is equal to the best IGP metric to the tunnel endpoint, regardless of the LSP. The tunnel metric is tunable using either relative or absolute metrics.

Because `autoroute` will use IGP metrics, you should ensure the native IP routing to prefixes you want to reach via the tunnel does not appear as "better". You can use `relative metric` set to `-2` to ensure the tunnel autoroute is always 2 less metric than native shortest-path. 


**Forwarding Adjacency**

The `forwarding adjacency` allows an operator to handle TE tunnel like another link on the IGP, based on SPF. 

By using forwarding adjacency, you can achieve the following goals:
- Better load balancing when you are creating POP-to-POP tunnels (ECMP)
- Use of tunnels from any upstream node, independent of the inner topology of the network

There are some restrictions:

- Increases the size of LSDB in the IGP.
- The link is advertised in the IGP network as a type, length, value (TLV) 22 object without any TE sub-TLV.
- TE tunnels must have bidirectional LSP 

However, MPLS TE with forwarding adjacency can advertise links between POP's to be equal-cost irrespective of the underlying IGP cost. This ensure POP-to-POP traffic is better load balanced.


**MPLS TE IPv6 Autoroute** advertises IPv6 routes over MPLS/TE IPv4 tunnels. The tunnel must be IPv6-enabled so it can carry IPv6 traffic. In order to advertise the tunnel, IPv6 forwarding adjacency or IPv6 autoroute announce must be configured on the tunnel.

Usage and restrictions of this...
- IPv6 autoroute feature is used for IPv4 MPLS TE tunnels using IPv6 routing
- The IPv6 traffic will not consider the UELB (Unequal Load Balancing) configuration. However, equal load balancing works for IPv6.
- IPv6 traffic will not consider policy-based tunnel selection, although it can be used for IPv4 traffic
- IPv6 autoroute announce and IPv6 forwarding adjacency are not supported by MPLS `auto tunnels`.


## Configure MPLS TE

To configure MPLS TE tunnels, MPLS should already be enabled on the core network.

The high level overview to conf MPLS TE is:

- Enable MPLS TE in the core
- Configure RSVP in the core
- Enable MPLS TE support in the core IGP OSPF or IS-IS
- Configure MPLS TE tunnels
- Configure routing into the the tunnels

### Enabling MPLS TE

``` py title="IOS XE"
router# conf t
router(config)# mpls traffic-eng
Router(config-mpls-te)# interface GigE 0/3
Router(config-mpls-te)# interface GigE 0/4
```

``` py title="IOS XR"
RP0/CPU0:RTR# conf t
RP0/CPU0:RTR(config)# mpls traffic-eng tunnels
RP0/CPU0:RTR(config)# interface GigE 0/0/0/4 
RP0/CPU0:RTR(config-if)# mpls traffic-eng tunnels
```

### RSVP Configuration

``` py title="IOS XR"
rsvp
!
  interface Gi0/0/0/4
    bandwidth 1000
  interface Gi0/0/0/1
    bandwidth 10000 1000
!
```

`bandwidth` `total-bandwidth max-flow` `sub-pool sub-pool-bw (optional)`

``` py title="IOS XE"
interface GigE 0/3
  ip rsvp bandwidth 10000 1000
!
```

### OSPF Configuration

``` py title="IOS XR"
router ospf 1
  mpls traffic-eng router-id Loopback0
  area 0
    mpls traffic-eng
  !
!
```

``` py title="IOS XE"
router ospf 1
  mpls traffic-eng area 0
  mpls traffic-eng router-id Loopback0
```


### IS-IS Configuration

``` py title="IOS XR"
router isis 1
  net 47.0001.0000.0000.0002.00
  address-family ipv4 unicast
    metric-style wide
    mpls traffic-eng level-1-2
    mpls traffic-eng router-id Loopback0
  !
!
```

``` py title="IOS XE"
router isis 1
  mpls traffic-eng level-1-2
  mpls traffic-eng router-id Loopback0
  metric-style-wide
```


### MPLS TE Tunnel Configuration

After enabling OSPF or IS-IS to distribute MPLS TE info, and after enabling MPLS TE functionality on desired interfaces + RSVP config, you can configure MPLS TE tunnel configs. 


[PE1] - [P1] - [P2] - [PE2]

PE1 router
``` py title="IOS XR"
interface Tunnel-te 1
  ipv4 unnumbered Loopback0
  signalled-bandwidth 1000
  destination 192.0.10.1 # this assigns an dest IP on the tunnel
  path-option 1 dynamic
  autoroute announce # this is autoroute
!
router static address-family ipv4
  unicast 192.0.100.0/24 tunnel-te 1 # this would be a static route option
```

PE2 router
``` py title="IOS XE"
interface Tunnel11
  ipv4 unnumbered Loopback0
  tunnel mode mpls traffic-eng
  tunnel mpls traffic-eng path-option 1
  tunnel mpls traffic-eng bandwidth 1000
  destination 192.0.2.1 # this assigns an dest IP on the tunnel
  tunnel mpls traffic-eng autoroute # autoroute 
!
ip route 192.0.200.0 255.255.255.0 Tunnel11 # this would be a static route
!
```

To configure an Explicit Cisco MPLS TE Tunnel (not dynamic path), do the following:

``` py title="IOS XR"
!
mpls traffic-eng
  reoptimize 300 # every 5 mins check
  interface Gi0/0
    admin-weight 55
!
interface GigabitEthernet 0/0
 description RED link to server farm, avoid having tunnels route through
 mpls traffic-eng attribute-flags 0x00000001
!
interface GigabitEthernet 0/1
 description BLUE link
 mpls traffic-eng attribute-flags 0x00000001
!
interface Tunnel-te 1
  description BLUE tunnel
  ipv4 unnumbered Loopback0
  signalled-bandwidth 2500
  destination 192.0.3.1
  priority 1 1
  path-option 1 explicit name Core2-3
  path-option 2 dynamic # if the static first path fails, a dynamic path LSP is found
  affinity 0x00000002 mask 0x00000002
  autoroute annouce # autoroute with forward adjacency
!
explicit-path name Core2-3
  index 1 next-address ipv4 unicast x.x.x.x
  index 2 next-address ipv4 unicast y.y.y.y
  index 3 next-address ipv4 unicast z.z.z.z
!
```



## MPLS TE Verification and Troubleshooting

These are IOS-XR...

`show mpls traffic-eng tunnel` to look at the status of the LSP tunnels and RSVP processes along with each tunnel and the hop list of current LSP.

`show mpls traffic-eng topology` will show neighbors on which interface, then different priorities and how much BW is allocated per priority + also the IGP and area.

**The first thing** to do when verifying MPLS TE would be to verify RSVP

`show rsvp session` and `show rsvp interface`

Then check the tunnel statuses etc.

To look at the routing table for MPLS TE routes, there is no unique table. You'll need to pipe to filter down.

`show ip route | include tunnel-te` if autoroute is enabled.


## MPLS TE Redundancy Mechanisms

Cisco MPLS TE tunnels can also be protected by backup tunnels to ensure speedy convergence in case of node or link failures. We can build parallel end-to-end tunnels, have backup tunnels on core nodes to protect individual links or neighboring nodes in case of failures.

Overall, the idea is that there are presignaled LSP already available if first tunnel fails. As soon as first tunnel fails, traffic is moved into second tunnel. Then back once first tunnel is re-available.

``` py title="IOS XR"
# at Dallas POP, tunnels to ATL
!
interface tunnel-te 1
  description Static tunnel to Atl
  ipv4 unnumbered Lo0
  destination 192.0.100.1
  signalled-bandwidth 250
  priority 1 1
  path-option 1 explicit name Core3-2
!
explicit-path name Core3-2
  index 1 next-address ipv4 unicast P1
  index 1 next-address ipv4 unicast P2
  index 1 next-address ipv4 unicast PE-ATL
!
interface tunnel-te 2
  description Dynamic backup tunnel to Atl
  destination 192.0.100.1
  signalled-bandwidth 125
  priority 2 2
  ipv4 unnumbered Lo0
  path-option 1 dynamic
!
router static address family ipv4
  unicast 192.0.200.0/20 tunnel-te 1 10
  unicast 192.0.200.0/20 tunnel-te 2 11
!
```

There is a drawback with backup tunnels, they double reserve bandwidth via RSVP over the entire path.

### Fast reroute (FRR)

Provides link protection to LSPs enabling the traffic carried by LSPs that encounter a failed link to be rerouted around the failure.

There is `link protection` and `node protection`.

The reroute decision is controlled locally by the router connected to the failed link. The headend router on the tunnel is notified of the link failure through IGP or through RSVP. When it is notified of a link failure, the headend router attempts to establish a new LSP that bypasses the failure. This provides a path to reestablish links that fail, providing protection to data transfer. The path of the backup tunnel can be an IP explicit path, a dynamically calculated path, or a semi-dynamic path. 

These tunnels are referred to as next-hop (NHOP) backup tunnels because they terminate at the LSP’s next hop beyond the point of failure. 

When the protected link or protected router go down, IGP re-converges. RSVP sends a message with session attribute flag 0x01=ON. (This means, Do not break the tunnel; you may continue to forward packets during the reoptimization.)

~50ms re-route protection.

Headend is notified via RSVP PATH_ERR and by IGP, there is a special flag in RSVP message that indicates the path must not be destroyed but continue an re-establish along new route.

``` py title="IOS XR on Dallas POP PE1"
!
interface tunnel-te 1
  destination 192.0.100.1 # this is to ATL POP PE1
  fast-reroute
  signalled-bandwidth 125
  priority 1 1
  ipv4 unnumbered Lo0
  path-option 1 dynamic
  autoroute annouce
  autoroute metric absolute 1
!
```

We manually craft the backup tunnel on intermediate core routers used by headend routers and FRR.

``` py title="IOS XR on Dallas P1"
!
interface tunnel-te 1001
  destination 192.0.80.1 # this is to P1 in ATL
  signalled-bandwidth 1000 # needs higher bw to support multiple tunnels
  priority 7 7
  ipv4 unnumbered Lo0
  path-option 1 explicit name Backup-path
!
explicit-path name Backup-path
  index 1 next-address ipv4 unicast ATL-P2
  index 2 next-address ipv4 unicast ATL-PE2
!
mpls traffic-eng 
  interface Gi0/0/0/1 # interface to P2
    backup-path tunnel-te 1000
```

### MPLS TE Autotunnel Backup

Configuring backup static tunnels for all core links does not scale well.

Autotunnel backup characteristics:

- Automates configuration of backup FRR tunnels for link and node protection.
- Configured globally and applies to all interfaces enabled for Cisco MPLS TE.
- Provides FRR for all tunnel LSPs traversing the protected interface.
- Autotunnel backup feature creates two types of link protection tunnels: 
    - Link-protection tunnels (NHOP protection)
    - Node-protection tunnels (next-next hop protection)

``` py title="IOS XR from Cisco.com"
Router# configure
Router(config)# mpls traffic-eng 
Router(config-mpls-te)# interface HundredGigabitEthernet 0/0/0/3
Router(config-mpls-te-if)# auto-tunnel backup
Router(config-mpls-te-if-auto-backup)# attribute-set ab
Router(config-mpls-te)# auto-tunnel backup timers removal unused 20
Router(config-mpls-te)# auto-tunnel backup tunnel-id min 6000 max 6500
Router(config-mpls-te)# commit
```

IOS-XR: `show mpls traffic-eng auto-tunnel backup summary`
IOS-XE: `show mpls traffic-eng auto-tunnel backup`

### MPLS TE Autotunnel Mesh

Manual configuration of tunnels could suck. This feature lets you setup a full mesh of TE tunnels automatically.

Autotunnel mesh characteristics:

- Simplify wide-scale deployment of Cisco MPLS TE tunnels.
- All routers configured with the same mesh group ID will form a full mesh of tunnels.
- Prefix list can define mesh-group member addresses.
- Attribute sets enable reusable configuration.

Auto-Tunnel mesh configuration minimizes the initial configuration of the network. You can configure tunnel properties template and mesh-groups or destination-lists on TE LSRs that further creates full mesh of TE tunnels between those LSRs. It eliminates the need to reconfigure each existing TE LSR in order to establish a full mesh of TE tunnels whenever a new TE LSR is added in the network. 

``` py title="IOS XR from Cisco.com"
Router# configure
Router(config)# mpls traffic-eng
Router(config-mpls-te)# auto-tunnel mesh
Router(config-mpls-te-auto-mesh)# tunnel-id min 1000 max 2000
Router(config-mpls-te-auto-mesh)# group 10
Router(config-mpls-te-auto-mesh-group)# attribute-set 10
Router(config-mpls-te-auto-mesh-group)# destination-list dl-65
Router(config-mpls-te)# attribute-set auto-mesh 10
Router(config-mpls-te-attribute-set)# autoroute announce
Router(config-mpls-te-attribute-set)# auto-bw collect-bw-only
Router(config)# commit 
```

We want to:

1. Configure attribute set
2. Enter automesh ID range and timers
3. Configure a mesh group ID with a prefix-list that defines included routers

A prefix-list can be configured on each TE router to match a desired set of router IDs. Then if you want to add a new router to the autotunnel mesh, add it into the prefix-list (or number it with an IP within the defined range). This enables provisioning requirement only on the added router, not all other routers too.


### MPLS TE End-to-End Path Protection

Path protection provides an end-to-end failure recovery mechanism for MPLS-TE tunnels. A secondary Label Switched Path (LSP) is established, in advance, to provide failure protection for the protected LSP that is carrying a tunnel's TE traffic.

Path protection characteristics:

- Protection of a Cisco MPLS TE path is provided end-to-end.
- Primary is the regular LSP, while protected is the backup LSP.
- Both tunnels are signaled, including protected to minimize on the failover time.
- Headend is notified of the failure through RSVP, BFD, or IGP.
- Path protection and FRR can be configured on the same tunnel at the same time with these benefits:
    - Protection is expanded.
    - Quick and effective re-optimization.
    - Total time on backup is reduced.

Although not as fast as link or node protection, presignaling a secondary LSP is faster than configuring a secondary primary path option, or allowing the tunnel source router to dynamically recalculate a path. The actual recovery time is topology-dependent, and affected by delay factors such as propagation delay or switch fabric latency.

Effectively, path protection switch over replaces the post-FRR LSP down event reoptimization

``` py title="IOS XR from Cisco.com"
# A mesh of R1-R2-R3-R4, here we connect R1 - R4 tunnel, then protected backups.
Router # configure
Router(config)# interface tunnel-te 0 
Router(config-if)# destination 192.168.3.3
Router(config-if)# ipv4 unnumbered Loopback0
Router(config-if)# autoroute announce
Router(config-if)# path-protection
Router(config-if)# path-option 1 explicit name r1-r2-r3-00 protected-by 2
Router(config-if)# path-option 2 explicit name r1-r2-r3-01 protected-by 3
Router(config-if)# path-option 3 explicit name r1-r4-r3-01 protected-by 4
Router(config-if)# path-option 4 explicit name r1-r3-00 protected-by 5
Router(config-if)# path-option 5 explicit name r1-r2-r4-r3-00 protected-by 6
Router(config-if)# path-option 6 explicit name r1-r4-r2-r3-00 protected-by 7
Router(config-if)# path-option 7 dynamic
Router(config-if)# exit
Router(config)# commit
```

### BFD for MPLS TE LSP

BFD is quicker, lighter, than LSP ping messages. Helps tear down tunnel-te interfaces more quickly in failure conditions to prevent traffic black holing and detect link failures more quickly.

``` py title="IOS XR"
interface tunnel-te 10
  bfd
    multiplier 5
    fast-detect
    minimum-interval 200
  !
!
```
