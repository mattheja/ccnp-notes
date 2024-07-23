# SR Traffic Engineering

## SR TE Concepts, Components, and Comparisons to RSVP TE

SR-TE uses policy to steer traffic. An SR-TE policy path is expressed as a list of segments that specifies the path, called a segment ID (SID) list.

Each segment is an e2e path from src to dst and instructs the routers in the network to follow the specified path instead of the shortest path calculated by the IGP. If or when a packet is steered into an SR-TE policy, the SID list is pushed onto the packet by headend. The rest of network executes the instructions in the SID list.

There are two types of SR-TE policies: explicit and dynamic.

### Differences from RSVP TE

TE state in RSVP is maintained by all nodes in the path by exchanging messages. In **SR-TE**, the state is only kept at the headend.

SR-TE is designed with SDN in mind. A signaling protocol is unnecessary, so this scales better.

The SR-TE model and SID list allow for IGP ECMP.


## SR Policy Concepts, Constraints, Metrics, and Attributes

SR-TE offers a rich set of options to govern traffic engineering with metrics and constraints. Aka... knobs.

The main components of SR-TE:

**Path** determination: 

- Who calculates the path? `Local` or `Central` -- done by headend node or PCE?
- Based on what? `Metrics` (cost, latency, bw, etc) or `Constraints` (color, affinity, node, etc)

**Steering** determination:

- Who determines the policy? `Local` or `Central`
- How is the policy created? static, autoroute, BGP SR-TE dynamic, etc


SR-TE uses **policy** to steer traffic through the network.

- Avoids the use of tunnel in context, since most policies do not require tunnel-te interfaces
- Tunnel interfaces deprecated
- SR-TE policy path is expressed as a SID-list 
- A packet is steered into SR-TE policy, then a SID list is pushed onto the packet by headend, while the rest of the network executes the instructions in the SID list (source routing)


**Binding segment** is fundamental building block. It is a local segment identifying a SR-TE policy. Each SR-TE policy is 1:1 with a binding SID.

- Binding SID is used to steer traffic into a SR-TE policy
    - The instruction associated with a binding SID is "POP and steer into this SR-TE policy"
- The Binding SID is a local label, statically or dynamically allocated for each SR-TE policy...
    - they can be manually configured, per policy in some cases
    - they can be allocated for RSVP-TE tunnels 
    - they are allocated when SR-TE policy is instantiated 
    - if you bring down SR-TE policy, binding SID remains

SR-TE offers comprehensive support for all useful optimizations and constraints:

- Latency
- Bandwidth
- Disjointedness
- Resource avoidance
- Weighted balancing

Then it can use 

- IGP metric
- TE metrics: `administrative distance`
- Delay as link-delay metric automatically measured by a node for attached links and distributed into IGP

**Margins**: can be configured to relax the "absolute" Min-Metric objective to favor more ECMP behavior.

- Headend builds SID list such that packets flowing through it do not use a path whose cumulated optimized metric is larger than the shortest path for the optimized metric + margin.
- Margin can be expressed as an absolute value or as a relative value (percentage) (margin relative <%>).


**Constraints** used by SR-TE:
- TE affinity
- IP addresses
- SRLG (??)
- Max accumulated metric (IGP, traffic engineering, and delay)
- Max number of SIDs in the solution SID list.
- Disjoint from another SR policy in the same association group

### Distribution of policies

SR-TE can be deployed locally or centrally. "Centralized" does not necessarily mean a complete state-of-the-art SDN controller. It may be simply a conventional stateful path computation element (PCE) running on a physical or virtualized router.


## SR-TE Traffic Steering

The path computation element protocol (PCEP) describes a set of procedures by which a path computation client (PCC) can report and delegate control of head-end label switched paths (LSPs) sourced from the PCC to a PCE peer.

The stateful model also enables a PCC to ask a PCE to perform computations to perform network-wide orchestration.

The binding SID (BSID) is the primary TE mechanism

- Locally programmed via BGP SR-TE dynamic:
    - destination or flow based
- Remotely programmed
    - "nesting" or "stiched" SR-TE policies
- Classic mechanisms with static routes, autoroute, PBTS (?), others can be used but not preferred (also likely half broken)

The BSID allows for

- Scalability
- Multidomain support
- Service-aware
- Data plane performance
- Loop-free network path
- Binding SID acting as the encapsulation key
- Automation and simplification using On-Demand Next Hop (ODN)

Policy instantiation is much easier compared to RSVP. Created an on-demand policy with a specific color, and it can dynamically generate a path to specified endpoint, such as L3VPN.

### BGP SR-TE Steering

Summary and examples...

Trigger automatic SR-TE policies for traffic to BGP destination
- Policies to meet SLAs, latency-optmized or other etc

Set at egress PE by adding a color-extended community to the router:
- color extended communitiy propagated to ingress PE
- traffic steering on ingress PE occurs automatically, based on the color, no routing policies in BGP required

Because the SID list associated with the policy is programmed into the hardware, there are no software-based recursive lookups causing a delay in the forwarding of traffic.

`show bgp` table can show `Colored Prefixes` in the next-hop column.

`show bgp x.x.x.x` will show `SR Policy Color` `BSID` associated as well.


## SR PCE-Based Paths

SR-TE was built around a SDN-like or SDN controller computing TE paths and communicating policies to headend routers.

The PCE does:

- **Compute path** and optionally...
- **Steer traffic** into policy

The PCC's can be stateful or stateless connecting with the PCE.

**PCEP** is the protocol/communication channel between the PCE and PCC.

If a PCE is operating as stateful, it has an additional database called **LSP DB**, while in both stateful and stateless modes, it has the **TED** or traffic engineering database.

The headend router can connect with the PCE to request a SR-TE policy for a LSP based on metrics and constrains provided by the PCC. If the PCC requires further changes to the policy, it will re-request the PCE to compute a new path, but each transaction is a single transaction. This is the `stateless` model. 

`Stateful` PCE maintains an active topology of the network via BGP-LS, where the PCC reports to the PCE any topology changes. 

The path computation element protocol (PCEP) describes a set of procedures by which a path computation client (PCC) can report and delegate control of head-end label switched paths (LSPs) sourced from the PCC to a PCE peer.

<br>

*(these definitions are from PCEP RFC 5440)*

**PCReq**: a PCEP message sent by a PCC to a PCE to request a path computation.

**PCRep**: a PCEP message sent by a PCE to a PCC in reply to a path computation request.  A PCRep message can contain either a set of computed paths if the request can be satisfied, or a negative reply if not.  The negative reply may indicate the reason why no path could be found.

In the stateful PCE setup process:

1. The PCC sends PCRequest
2. PCE calculates path
3. PCE sends PCReply
4. PCC sets SR-TE policy
5. PCC sends PCReport
6. PCE updates LSP DB

