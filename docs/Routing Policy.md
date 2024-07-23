# Routing Policy and Manipulation

## Routing Protocol Tools 

For security and performance reasons, you will always use advanced filtering and policy mechanisms with BGP.

Filtering can be performed at these points:

- *Incoming* updates as they are received from neighbors but before they are installed into protocol and RIB
- *Outgoing* updates before they are sent to neighbor
- *Redistributed* updates from other protocols

You filter routing info based on:

- Prefix and prefix length
- Then update parameters (attributes) that are specific to that protocol

Then filtering uses these tools in routing software:

- Prefix lists:
    - Used for prefix-based filtering or route matching
- AS path access lists:
    - Used in BGP for filtering or route matching based on BGP AS path attribute
- Route maps:
    - Implements complex routing policies
    - Can also be used to filter 
- RPL (Route Policy Language):
    - Replaces route maps on IOS XR, more complex feature rich language

### Filtering Examples

So at a high level, we implement filtering because of:

1. Security
2. Cost
3. Performance

And then filter:

- Particular Routes
- Specific characteristics of the routes


Different protocols perform filtering very differently... 

#### Filtering in OSPF:

- Prefix and prefix length
- LSA type (internal, external, NSSA-external)
- route source

***Done on ABR or ASBR's***

Filtering on ASBR for redistributed routes:

- Static and connected routes
- Routes from other OSP processes
- Routes from other routing protocols

Filter on ABR against interarea routes using prefix lists on which routes can be shared between areas.


#### Filtering in IS-IS:

Similiar characteristics as OSPF. Except you can use prefix lists, route maps, or routing policies to filter an exchange of routing info between IS-IS levels

- Filter on Level 1-2 routers
    - Filter L1 to L2 routes.
    - Conditional L2 to L1 route leaking.
- Filtering on redistributing routers (any level) to filter external routes.


#### Filtering in BGP:

You will almost always/should/must filter in BGP. The type and direction depends on the neighbor (internal, customer, or external).

Inbound filtering examples:

- permit in only customer routes for end customers
- permit in a specific list of routes from subordinate SP's, SP's that are peering at an exchange
- permit the complete inet routing info between upstream service providers

Outbound filtering examples:

- permit only the default route with single-homed customers
- permit default route and local routes, such as for multihomed customers using this service as a backup route but want access to local destinations
- Permit all routes

Commonly filtered on attributes with BGP:

- Prefix and prefix length
- Next-hop attribute based on BGP next-hop address
- Route source address
- AS path attribute
- Local preference
- BGP communities 


Some of the most commonly implemented policies are:

- Customers using AS path prepending to make one path less desirable than another when multihomed
- Customers can alternatively signal to their upstream providers prefernces using BGP communities. The upstream SP will then translate these communities received to some BGP attribute (like AS path prepending or local preference).
- SP's can use BGP local preference to influence route selection internally to prefer one path over another

## AS Path-based filtering

IOS/XE supports this with AS path access lists. IOS-XR supports similiar filtering with RPL.

### IOS/-XE Configurations

Uses AS path access lists, then something is done with this.

The syntax is:

- use a unique number.
- regex used to match based on the contents of the AS path attribute.
- AS path attribute processed as a string of characters.

`ip as-path access-list [acl-number] {permit|deny} [regex string with AS numbers]]`

An example like `ip as-path access-list 1 permit ^$` matches any route that has an empty AS path attribute. Only locally originated routes have an empty AS path attribute, so this will only send local originated routes. This is one method to prevent accidental transit AS. 

## Route maps

Route maps are a relatively simple language to support complex policies + filtering

Some characteristics:

- Allow you to write complex policies
- Uniquely identified with case-sensitive names
- Each route map consists of one or more statements
- Each statement contains zero or more `match` commands
- Each statement contains zero or more `set` commands used to modify routing updates

Route map processing:

1. Match y/n? 
2. If Match, then permit y/n?
3. If permit, then set y/n?
4. If permit yes or no, then send

Route maps have an implicit deny at the end. If there are no matches, then routes are dropped.

### Syntax
```
route-map (map-tag) [permit|deny] {sequence number}
  match (condition)
  match (condition)
  set (parameter-value)
  set (parameter-value)
```

If 2 match statements, then they are evaluated as such:

- match conditions of same type are evaluated using logical `OR` operator 
- match condtions of *different* types are evaluated using logical `AND` operator

If there is no `match` command, then it matches anyways.

There are additional route map options:
- `continue` to jump to another statement instead of exiting or impled continue if no sequence number specified. (More here)[https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-route-map-continue.html]
    - Useful when gathering policy (matches and sets) into a single phrase or some more logical constructs (can't think of any examples...)
- `policy-list` in match statements, like `match policy-list (other-route-map)` which has a list of permit lines with matches and other logic

To implement a permit all logic, you would do something like:
```
route-map Example_Peering_Ngbh permit 10
 match ip address prefix-list preferred_prefixes
 set local-preference 200
!
route-map Example_Peering_Ngbh permit 1000
```

## Routing Policy Language

RPL has these characteristics:

- Replaces route maps in IOX-XR
- Designed for large-scale routing configs
- "Simple", powerful language, designed to process routing updates
- Addresses the deficiencies of route maps...
    - Better modularity and reusability
    - Parameterization
    - Nesting of policies and conditions
    - Greater matching and reusable value sets

Has the notion of sets: prefix-sets, community-sets, as-path-sets, extcommunity-sets, and rd-sets that represent groups of IPv4/IPv6 prefixes, etc, route distinguishers values. 

### RPL Syntax

Each routing policy is defined with a case-sensitive name. The policy is defined within `route-policy $name` and `end-policy`. 

A very simple "allow all" policy:
```
route-policy PermitAll
  pass
end-policy
```

There is a **process for pass/drop**:

- default action is drop, so an empty policy implicitly denies all
- an explicit `pass` by itself will forward all routes without modification
- an explicit `drop` by itself will deny/dropp all, even if there are other statements preceding or other statements in the policy
- a `set` command will modify attributes accordingly and pass all routes

RPL has a number of operators to compare attributes:

`eq` - attribute numerically equal to specified value
`le` / `ge` - attribute is numerically lower/greater or equal to the specified value
`is` - attribute is equal to specified value. *use* this for nonnumerical values
`in` - attribute is contained in a value set

Then RPL has many attribute-specific operators, like AS path matching.

Also basic boolean operators: `AND`, `OR`, `NOT`

An example:
```
route-policy transit-LocalPref
  if med eq 10 and not local-preference eq 100 then
    set local-preference 300
  elseif med eq 20 or local-preference eq 200 then
    set local-preference 200
  else
    set local-preference 150
  endif
end-policy
```

There are two types of nesting characteristics: 

1. Nesting conditional statements
```
route-policy OuterRP
  if med eq 10 then
    if local-preference eq 200 then
      set community (1:10) additive
    endif
  endif
end-policy
```

2. Nested hierarchical 
```
route-policy Set10C
  if local-preference eq 100 then
    set community (1:10) additive
  endif
end-policy


route-policy MatchMED
  if med eq 10 then
    apply Set10C
  endif
end-policy
```

### RPL Attributes and Parameters

Within RPL, to assign values to attributes and parameters, use `set`. There are some intricacies that are important when building RPL:   

All `set` statements are processed upon completion of the route-policy.

The last applied `set` wins if they match against the same unique parameter, such as local preference

For non-unique parameters, like AS-path, all `set` commands are evaluated in order

When working with communities, remember that:

- You can assign one or more values to the BGP community attribute
- If you use `additive`, new communities are added to existing BGP community attribute
- If you do not use `addititive, then existing BGP communities are overwritten 
- `delete` let's you delete some or all BGP community attributes

Some other commonly used `set` commands with BGP:

`set local-preference [*|+|-] (value)` - syntax for setting local-pref

`set med {[+|-] value | igp-cost | max-reachable}` - for setting multi-exit discriminator (med)

`set dampening [halflife value] [max-suppres value] [reuse value] [suppress value]` - route-flap dampening

`prepend as-path {AS | most-recent | own-as} [count]` - AS-path prepending

`suppress-route` - used when doing route aggregation, will suppress if aggregated

`unsuppress-route` - same as above, will unsuppress if agg

### OSPF and IS-IS attributes and parameters using RPL

These OSPF parameters can be modified with `set`:

`set metric-type {type 1|type 2}` to change the OSPF metric type
`set ospf-metric [value]` to modify the OSPF metric
`set tag [value]` to tag OSPF routes

These IS-IS parameters can be modified:

`set metric-type {external|internal}` to change IS-IS metric type 
`set isis-metric [value]` to set the IS-IS metric
`set level {level-1 | level-2 | level-1-2}` to set IS-IS level for redistributed routes
`set tag [value]` to tag IS-IS routes


### RPL Parameterization

Use parameters to make your RPL configs modular and more readable. You can re-use parameters globally and within individual policies/nested policies.

You can also "pass" parameters into a nested routing policy, both using global parameters and nested routing policies

Global parameters are defined with `policy-global`, and they'll be available for all policies to use.

If there is a global and local parameter name "collision" then the local parameter takes precendence.

```
policy-global
  # global vars
  AS '65001',
  Lo0 '10.100.70.1',
  EBGP1 '192.168.2.1',
  EBGP2 '192.168.3.1',
  upstream-weight '5',
  upstream-LP '100',
  DefMED '0'
end-global
```
```
route-policy Set-Upstream
  if as-path originates-from '$AS' then
    set med $DefMED
  endif
end-policy
```

#### Characteristics of the RPL passed parameters:

- Declare parameters when creating a routing policy.
- Nesting policies with parameters allows for greater modularization and optimization of policies.

```
route-policy SetMED($med,$as)
  if as-path originates from '$as' then
    set med $med
  else
    set med max-reachable
  endif
end-policy
!
```
```
route-policy ProcessUpdates
  if as-path neighbor-is '1299' then 
    apply SetMED(50,1299)
  elseif as-path neighbor-is '6939' then
    apply SetMED (100,6939)
  endif
end-policy
!
```


### Value Sets

In general programming sense, a `set` is an unordered collection of unique elements. 

RPL provides sets as containers for groups of values for matching purposes. 

- Used in conditional expressions for matching
- You can create inline sets for one-time use
- You can create named sets for reusability

There are 5 kinds of sets:

1. AS-path set `as-path-set`
    - An AS path set can contain one or more regex paths, like `ios-regex ^7922_10999`
2. Community set `community-set`
3. Extended community set `extcommunity-set`
4. Prefix set `prefix-set`
5. Route distinguisher (RD) set `rd-set`

In-line set example:
```
route-policy RP-inline-example
  if attribute in (value1, value2, ...)
then
  set local-preference 150
endif
end-policy
```

Named set example:
`xy-set` is stand-in for any of the various types
```
xy-set set-name
  value1,
  value2
end-set
!  
route-policy RP-named-example
  if attr in set-name then
    set local-preference 200
  endif
end-policy
```

**AS-path Sets**

- Can contain one or more AS path regex

```
as-path-set OutboundAvoids
  ios-regex '_10$',   # originates-from path
  ios-regex '^6939_',  # neighbor-is path
  ios-regex '_174_',  # passes-through ASN
  ios-regex '^3356_'  
!
route-policy RP
  if as-path in OutboundAvoids then
    set local-preference 90
  endif
end-policy
```

Regex can be resource expensive, so alternatively, there are predefined AS path matching rules

`neighbor-is path`: matches based on first ASN in the AS path, same as regex '^path_‘

`originates-from path`: matches based on last ASN in the AS path, same as regex '_path$'

`passes-through ASN`: matches based on ASN anywhere in the AS path, same as regex  '\_path\_‘)

`length len`: matches AS paths based on number of ASNs in the path

`unique-len len`: matches AS paths based on number of unique ASNs in path

- These are used as condition statements in RP, like 

```
if as-path originates-from '100' or as-path originates from '200' then
  set local-preference 70
```


**Standard Community Sets**:

A community set holds community values for matching against standard community values, which must be split in half and expressed as two unsigned decimal integers 0-65535, seperated by a colon. 

Characteristics:

- Defined as a global set with `community-set`
- Use one or more comma-seperated match options in CLI
    - `ios-regex` to define regex set membership
    - numbered membership matching
    - membership matching against well-known standard communities
- use `match-any` operator to match routes that have at least one community in community set
- use `match-every` operator to match routes where every element of community set matches a community from the route
- `matches-within` operator to match routes where every community *from the route* matches an element within the set

```
community-set MyComms
  ios-regex '64998:10..'
  ios-regex '64998:20..'
end-set
```
```
community-set MyComms2
  ios-regex '64998:[12]0..'
end-set
```
```
route-policy Comm2LP
  if community matches-any MyComms then
    set local-preference 200
  endif
end-policy
```

Both examples `community-set`'s do the same thing. Also, ranges and wildcards can be used with community-set sets, like:

`ASN:num`
`ASN:[range]`
`ASN:*`

Also, well-known communities has identifiers:

`internet`: matches all communities *I am still not entirely certain what this one is... the others are in RFC1997 but not this one?*
`local-as`: keeps tagged prefixes in the local AS
`no-advertise`: prevents tagged prefixes from being advertised to any peer
`no-export`: prevents tagged prefixes from being announced to EBGP peers

like:
```
router bgp 64998
  address-family ipv4 unicast
    redistribute connected route-policy NoExport
!
route-policy NoExport
  set community no-export
end-policy
!
```


**Extended Community Sets**:

Same as standard community sets, except contains support for extended communities. It also supports named forms and inline forms. Generally there are 3 types of extended communities sets: `cost`, `soo`, `rt`.

Characteristics:

- `extcommunity-set {cost | rt | soo}` command
- Then use one or more comma-separated match options with `ios-regex` or numbered community membership matching
- use `match-any` operator to match routes that have at least one community in community set
- use `match-every` operator to match routes where every element of community set matches a community from the route
- `matches-within` operator to match routes where every community *from the route* matches an element within the set

```
extcommunity-ext rt RtExtCommPolicy
  ios-regex '6500[1..9]:10..',
  65000:*
end-set
!
route-policy ExtCommPolicy
  if extcommunity rt matches-any RtExtCommPolicy then
    set local-preference 200
  endif
end-policy
```

A cost set is an extcommunity set used to store cost EIGRP Cost Community type extended community type communities
An rt set is an extcommunity set used to store BGP Route Target (RT) extended community type communities
A soo set is an extcommunity set used to store BGP Site-of-Origin (SoO) extended community type communities


**Prefix sets**:

A prefix-set holds IPv4 or IPv6 prefix match specifications. The most simple to understand. 

```
prefix-set PrivatePrefixes
  10.0.0.0/8 le 32
  172.16.0.0/12 le 32
  192.168.0.0/16 le 32
end-set
!
prefix-set Core-loopbacks
  172.16.1.0/24 eq 32
end-set
```


**rd set**:

An rd-set is used to create a set with route distinguisher (RD) elements. An RD set is a 64-bit value prepended to an IPv4 address to create a globally unique Border Gateway Protocol (BGP) VPN IPv4 address.

```
rd-set rdset1
     10.0.0.0/8:*,
     10.0.0.0/8:777,
     10.0.0.0:*,
     10.0.0.0:777,
     65000:*,
     65000:777
  end-set
```


## Routing Policies Application, Maintenance, Monitoring, and Testing

When building routing policy, it is important to clearly define requirements. Once business and long-term strategy have been defined and reviewed, you should use these 4 steps to create and apply policies:

1. design routing policy
2. configure routing policy (in a test env first as needed etc)
3. test the policy, using `show` etc
4. apply the policy if correct meeting expectations

When applying route policies, you have to think about the specific routing protocol mechanics and where you can attach the policy. Such as:

- redistribution between any two protocols
- received or sent updates, depending on the limitations of routing protocols (i.e.-- OSPF and ABR's)
- origination of routes in BGP by using network statements or summarization
- injecting routes into the routing table from BGP

On IOX-XR, when you configure and commit changes, it performs a validity check in two phases:

1. basic syntax and value checking when actually hitting enter upon making changes
2. then once you commit, policies and changes verified for a given attach point
  - for example, if you configure OSPF metrics in a route-policy, you cannot attach it to a policy used for BGP neighbor-in filtering


`show rpl` or `show rpl <$policy> detail` or `show rpl <$policy> attachpoints`.

`show bgp route-policy <$policy>` can show filtered output for policies 

### Maintaining route policies

You cannot edit the routing policy directly with the CLI commands.

- Use available editors through exec mode, like `route-policy Upstream-BGP-IN` to create and being editing.
```
RP/0/RP0/CPU0:P1# edit route-policy Upstream-BGP-IN ?
  emacs  to use Emacs editor
  nano   to use nano editor
  vim    to use Vim editor
  <cr>
```


## Route Maps to RPL

You generally can map each numbered entry to an `if` statement, then each route-map `match` statement is an conditional (`if`, `elif`, `else` etc)

Route maps with 2 `match` conditions of same type should be joined into a conditional with `OR` operator

Route maps with 2 `match` conditions of different types should be joined into a condition with `AND` operator

