# Answers to Chapter Questions

Answers and short explanations for the **Check your understanding** questions at the end of each chapter. Try the questions first; each section links back to them.

## Chapter 1: The Data Center Network Problem

**1. C. The access layer**

Servers connected to access switches; aggregation bundled the access switches, and the core joined the aggregation blocks. Spine is a spine-leaf term, not a three-tier one.

**2. C. North-south traffic between servers and outside clients**

Three-tier assumed most traffic entered at the core, went down to a server and back out: north-south. The other three are all server-to-server, east-west flows.

**3. B. East-west flows between blocks had to climb to the core and come back down**

Every flow between two blocks crossed the core, so the core had to carry all of it; making it faster meant ever bigger chassis. Spanning tree and VLANs were separate problems.

**4. D. 6:1**

48 x 10 = 480 Gbps down and 2 x 40 = 80 Gbps up, and 480 / 80 = 6, so 6:1.

**5. B. Blocks all but one so the topology has no loops**

STP finds a loop-free tree by blocking redundant links. The blocked link stays cabled but carries nothing until the active one fails, wasting capacity. Bundling is link aggregation, not STP.

**6. D. Adding capacity by adding more of the same small switches**

Scale-out adds more identical units. Replacing the core with a bigger box is scale-up, the approach that stopped working.

**7. A. A single large core switch for all traffic**

The fourth requirement is scale-out growth with small switches. One large core is exactly the scale-up design being replaced.

**8. C. Internal east-west flows fight for scarce uplinks, causing congestion and unpredictable latency**

Applications now depend on many internal calls between servers. When uplinks are scarce, those flows congest and latency becomes unpredictable.

[Back to the questions](01-dc-network-problem.md#check-your-understanding)

## Chapter 2: The Spine-Leaf CLOS Fabric

**1. D. Every leaf connects to every spine, and no leaf-to-leaf or spine-to-spine links exist**

Every leaf to every spine, and nothing else between switches. That rule is what gives every pair of leaves equal parallel paths.

**2. C. Three**

Each spine offers one leaf-spine-leaf path, so the number of paths equals the number of spines: three.

**3. D. Spreads traffic across all equal-cost paths at once**

Equal-Cost Multi-Path forwarding uses all equal paths simultaneously, so every link carries traffic.

**4. A. Three: a leaf, a spine and another leaf**

The path is always leaf, spine, leaf. That fixed length is why latency between any two servers is consistent.

**5. C. A spine**

A new spine gives every leaf one more parallel path, widening the whole fabric. A new leaf adds server ports, not leaf-to-leaf bandwidth.

**6. A. The number of uplink ports on each leaf**

A leaf connects once to each spine, so it can have only as many spines as it has uplink ports.

**7. B. Connecting several spine-leaf pods with another CLOS stage**

When one fabric is not enough, each fabric becomes a pod and a super-spine layer joins the pods, applying the same CLOS idea one level up.

**8. C. One configuration template covers every switch of a given role**

Uniform hardware per role means one template, filled with per-switch data, configures every spine or every leaf.

[Back to the questions](02-spine-leaf-fabric.md#check-your-understanding)

## Chapter 3: The IP Underlay

**1. A. No Layer 2 domain spans the fabric, so there are no Layer 2 loops to prevent**

With no Layer 2 domain stretched across the fabric, STP has nothing to block; the routing protocol handles loop prevention while using many paths.

**2. A. It holds exactly two usable addresses, one per end, and wastes nothing**

A /31 has exactly two addresses, enough for the two ends of a point-to-point link with no waste.

**3. B. It stays up as long as the switch is alive and is reachable over every uplink**

A loopback belongs to the switch, not to one link, so it survives link failures and is reachable over all paths. Later the VTEPs use it.

**4. C. RFC 7938**

RFC 7938 covers BGP in large-scale data centers. RFC 7348 is VXLAN, 7432 is EVPN, 8365 is EVPN over VXLAN.

**5. C. The same BGP sessions can later carry the EVPN overlay too**

One protocol and one set of sessions serve both the underlay and, later, EVPN. OSPF and IS-IS also support ECMP.

**6. A. BGP rejects a route whose AS path already contains its own ASN**

With a unique ASN per leaf and a shared ASN for the spines, a route never comes back to the leaf it came from, nor loops between spines.

**7. A. Installs up to 64 equal-cost paths so ECMP uses every spine**

Without it, BGP installs one best path and ECMP collapses to a single link. It must be at least the number of spines.

**8. C. Hand-assigned point-to-point /31 addresses, by peering over IPv6 link-local addresses**

Sessions form over IPv6 link-local addresses, so you no longer assign or track /31s per link. Loopbacks and ASNs remain.

[Back to the questions](03-ip-underlay.md#check-your-understanding)

## Chapter 4: Why Overlays

**1. B. They behave as if on the same Ethernet segment, same subnet, able to broadcast to each other**

Layer 2 adjacency is about sharing a broadcast domain and subnet, not physical placement.

**2. C. A moved VM expects to keep its IP address and default gateway**

A migrating VM keeps its IP and gateway, which only works if both hosts appear to be on the same segment.

**3. A. It brings back spanning tree, broadcast storms and the 4094-VLAN ceiling**

Stretching Layer 2 undoes the routed design of Part I and reintroduces its problems.

**4. B. The physical routed fabric that carries the tunnels**

The underlay is the routed spine-leaf fabric; the overlay is the virtual network built on top of it.

**5. C. Wraps the whole frame inside an IP packet addressed to the far tunnel endpoint**

Encapsulation keeps the original frame intact as the inner packet and adds an outer IP header the fabric can route.

**6. B. The underlay deals only with outer IP headers between switches; tenant changes stay in the overlay**

VMs come, go and move inside the overlay. The underlay only sees outer packets between switches, so it rarely changes.

**7. D. VXLAN**

VXLAN won in hardware fabrics. Geneve is gaining ground, especially in software; NVGRE and STT were early contenders.

**8. B. UDP ports let the fabric's ECMP hashing spread tunneled flows across spines**

ECMP hashes on ports among other fields; UDP gives VXLAN a port field to vary per flow. UDP does not encrypt or guarantee delivery.

[Back to the questions](04-why-overlays.md#check-your-understanding)

## Chapter 5: The VXLAN Data Plane

**1. A. The leaves**

Leaves sit between the servers and the fabric, so they encapsulate and decapsulate. Spines just route the outer packets.

**2. C. Its underlay loopback**

Tunnels run between loopbacks, which the underlay already reaches over ECMP. That is the tie between the two layers.

**3. B. 4789**

4789 is the IANA port for VXLAN. 179 is BGP, 6081 is Geneve, and 8472 is an older Linux default.

**4. C. 24**

24 bits gives about 16 million segments, against 12 bits and 4094 usable values for a VLAN ID.

**5. B. So different inner flows hash onto different spines, using all ECMP paths**

Without it, all traffic between two VTEPs would have identical outer headers and pin to one spine.

**6. C. The outer headers only: UDP between two loopbacks**

Spines route the outer IP packet and never look inside it.

**7. A. Each tenant's VLAN 10 is mapped to a different VNI**

VLAN IDs are local; the VNI is the fabric-wide segment number, and different VNIs are isolated.

**8. C. How a VTEP learns which remote VTEP holds a given MAC, and how to handle broadcasts**

The data plane forwards; knowing where MACs live is a control-plane question, answered first by flood-and-learn, then by EVPN.

[Back to the questions](05-vxlan-data-plane.md#check-your-understanding)

## Chapter 6: Flood-and-Learn VXLAN

**1. D. Broadcast, Unknown-unicast and Multicast**

BUM is the traffic a VTEP must flood: broadcasts, unicast to unlearned MACs, and multicast.

**2. B. Broadcast**

ARP requests are sent to the broadcast MAC address.

**3. D. The source VTEP, one unicast copy per remote VTEP in the VNI**

The source does all the work. It needs no fabric support, but the copies grow with the number of VTEPs.

**4. B. It requires running multicast routing (PIM) across the underlay**

Multicast is efficient at the source, but many operators avoid running PIM in the underlay for its complexity.

**5. C. Inner source MAC and outer source IP**

The inner source MAC lives behind the outer source IP, the sending VTEP's loopback.

**6. D. There is no control plane: all knowledge comes from watching traffic**

VTEPs never tell each other what they know, so every discovery needs flooding.

**7. B. Every host broadcasts ARP periodically, and each broadcast is flooded to every VTEP in the segment**

A steady rain of ARP broadcasts consumes bandwidth and CPU on every VTEP, even those that don't care.

**8. C. Let VTEPs use BGP to tell each other which hosts they have**

BGP already runs the underlay and exists to distribute reachability. Using it for host information is EVPN.

[Back to the questions](06-flood-and-learn.md#check-your-understanding)

## Chapter 7: EVPN Fundamentals

**1. B. A BGP-based control plane that distributes host and segment information**

EVPN is the control plane; VXLAN remains the data plane that forwards packets.

**2. D. RFC 8365**

RFC 7432 defined EVPN, RFC 8365 covers EVPN over VXLAN (network virtualization overlays), RFC 7348 is VXLAN itself.

**3. D. The ordinary way, from the data plane or ARP, then advertises them**

Locally attached hosts are still learned locally; the change is that the VTEP immediately advertises them in BGP.

**4. B. It keeps the number of BGP sessions linear in the number of VTEPs instead of quadratic**

A full mesh of n VTEPs needs about n squared sessions; with reflectors, each VTEP needs one or two.

**5. C. AFI 25 (L2VPN), SAFI 70 (EVPN)**

AFI 1 SAFI 1 is IPv4 unicast, used by the underlay. EVPN is L2VPN/EVPN, 25/70, often on the same sessions.

**6. B. To make each advertised address globally unique, even when tenants overlap**

The RD keeps overlapping tenant addresses distinct inside BGP. Import decisions are the Route Target's job.

**7. B. To control which routes a VTEP exports and imports**

An RT is an extended community; a VTEP imports only routes carrying an RT it is configured to accept.

**8. C. ARP suppression, answering ARP locally from learned IP-to-MAC bindings**

Because every VTEP knows remote IP-to-MAC mappings from EVPN, it can answer ARP locally. ECMP, VNIs and routed links existed already.

[Back to the questions](07-evpn-fundamentals.md#check-your-understanding)

## Chapter 8: EVPN Route Types

**1. B. Type 2**

Type 2, the MAC/IP advertisement, is the route you see most in show output.

**2. B. Type 3**

Type 3, Inclusive Multicast Ethernet Tag (IMET), builds the flood list per VNI.

**3. D. Type 5**

Type 5 is the IP Prefix route, covered in Chapter 10.

**4. A. Types 1 and 4**

Type 1 (Ethernet auto-discovery) and type 4 (Ethernet segment), covered in Chapter 11.

**5. B. The IP address paired with the MAC**

When the route carries both MAC and IP, every VTEP learns the binding and can answer ARP itself.

**6. B. The L2VNI, the bridging segment the host is in**

Label 1 is the L2VNI; the optional Label 2 is the L3VNI used for routing between segments.

**7. D. Manual or multicast-based BUM configuration: the flood list is built from BGP**

The flood list now builds and updates itself from BGP, so most fabrics use head-end replication without underlay multicast.

**8. B. Known unicast**

Known unicast uses type 2; BUM discovery uses type 3; inter-subnet routing type 5; multihoming types 1 and 4.

[Back to the questions](08-evpn-route-types.md#check-your-understanding)

## Chapter 9: EVPN-VXLAN in Action

**1. A. Builds a type 2 route for VM A and sends it to the spines, which reflect it**

Leaf 1 learns VM A locally and advertises it at once as a type 2 route; the spines reflect it to every leaf.

**2. C. Leaf 3 answers it from its EVPN knowledge; the broadcast never crosses the fabric**

This is ARP suppression: Leaf 3 already holds VM A's IP and MAC from the type 2 route.

**3. B. VNI 10, outer IP from Leaf 3's loopback to Leaf 1's loopback**

Exactly as in Chapter 5: VXLAN with the segment's VNI, outer IP between the two VTEP loopbacks.

**4. D. Discovery flooding, ARP flooding and the first-packet penalty**

The control plane delivered everything beforehand, so nothing had to be flooded.

**5. B. l2vpn evpn**

EVPN routes travel in the l2vpn evpn address family.

**6. D. Advertises the locally configured VNIs and their hosts into EVPN**

It tells BGP to advertise all locally known VNIs and the hosts learned in them.

**7. A. EVPN, not data-plane learning, is now the source of truth for where MACs live**

With EVPN driving the MAC table, data-plane learning would only compete with it.

**8. C. Route traffic between different subnets**

So far everything stays within one subnet and VNI. Routing between subnets comes in Chapter 10.

[Back to the questions](09-evpn-vxlan-in-action.md#check-your-understanding)

## Chapter 10: Routing in the Overlay

**1. D. All inter-subnet traffic funnels to one gateway leaf: a bottleneck and a long path**

Traffic would travel to the gateway's leaf, be routed there, and travel back.

**2. B. Every leaf configured with the same gateway IP and MAC for a subnet**

Every leaf answers to the identical gateway address, so a host's gateway is always its own leaf.

**3. D. The gateway IP and MAC are the same on every leaf, so a moved VM finds its gateway unchanged**

No relearning is needed: the same gateway is already on the new leaf.

**4. A. A routing segment, one per tenant VRF, carrying routed traffic between subnets**

L2VNIs bridge within subnets; the L3VNI is the transit VNI for routed traffic in a tenant's VRF.

**5. C. Every leaf must be configured with every subnet's L2VNI, even without local hosts in it**

The ingress leaf routes and then bridges in the destination's L2VNI, so it needs every L2VNI.

**6. D. In the tenant's shared L3VNI, routed on both ingress and egress leaves**

Ingress routes into the L3VNI, egress routes out of it. Each leaf needs only its local L2VNIs plus the L3VNI.

**7. B. The tenant VRF's L3VNI and a router MAC extended community**

The router MAC tells the receiving leaf which inner MAC to use on the transit VNI.

**8. C. Advertises external destinations into the fabric and fabric subnets outward**

Border leaves connect the fabric to the WAN and external routers, in both directions, using type 5.

[Back to the questions](10-overlay-routing.md#check-your-understanding)

## Chapter 11: Multihoming and Redundancy

**1. A. The set of leaf links that attach to the same multihomed server, identified by an ESI**

Both leaves configure the same ESI for the links to that server.

**2. C. The two leaves coordinate through BGP EVPN routes, with no special peer link**

Standards-based EVPN multihoming coordinates over BGP, so any two leaves can share a server.

**3. B. The one leaf on a segment that delivers BUM traffic to the server**

If both leaves forwarded BUM, the server would get duplicates, so only the DF does.

**4. C. Type 4**

Type 4, the Ethernet Segment route: leaves that see a shared ESI know they are peers and run the election.

**5. C. A server receiving its own flooded frame back through its other leaf**

A frame that came from a given Ethernet Segment is never sent back out that segment's local links.

**6. A. Letting remote VTEPs load-balance to both leaves even if they learned the MAC from only one**

The type 1 route tells remote VTEPs that both leaves reach everything on the segment.

**7. A. One per-ES type 1 route is withdrawn: mass withdrawal**

One withdrawal moves all MACs on the segment to Leaf 2 at once, instead of thousands of type 2 withdrawals.

**8. C. Many fabrics keep it simple: MLAG, or single-homed servers with application-level redundancy**

How much multihoming to use is a design decision; many servers are simply single-homed.

[Back to the questions](11-multihoming.md#check-your-understanding)

## Chapter 12: Designing a Fabric

**1. B. The number of uplink ports on each leaf**

A leaf connects once to each spine, so its uplink count caps the spines.

**2. A. The number of ports on each spine**

A spine connects once to each leaf, so its port count caps the leaves.

**3. B. 1.5:1**

1200 Gbps of server capacity against 800 Gbps of uplinks gives 1.5:1.

**4. C. About 3000**

64 leaves (one per spine port) times 48 servers is 3072.

**5. C. Build more fabrics as pods and join them with a super-spine stage**

Each fabric becomes a pod; a super-spine layer forms a 5-stage CLOS with the same small switches.

**6. B. People and scripts can read a switch's identity from its numbers, which makes the fabric automatable**

Regular, predictable numbering is what lets templates and scripts generate configurations.

**7. B. To connect the fabric to the WAN, internet edge, firewalls and load balancers**

They advertise external destinations in as type 5 routes and summarize fabric subnets outward, keeping other leaves uniform.

**8. B. Consistency: every switch of a role follows the same rule, removing a class of human error**

Templates depend on a numbering plan, and in return remove per-device hand configuration mistakes.

[Back to the questions](12-designing-a-fabric.md#check-your-understanding)

## Chapter 13: Operating and Troubleshooting

**1. A. Work from the bottom up: confirm the underlay before suspecting the overlay**

Each layer depends on the ones below, so a fault low down shows up as symptoms everywhere above.

**2. C. MTU: VXLAN adds about 50 bytes, so full-size frames are dropped after encapsulation**

Small pings fit; full-size frames plus VXLAN overhead exceed an MTU that was not raised.

**3. B. A session can be up for IPv4 unicast but never activated for l2vpn evpn**

An established IPv4 session says nothing about whether the EVPN family was activated.

**4. D. The broken VNI's mapping and RT import, so its type 2 and type 3 routes arrive**

If one VNI works, the underlay and sessions are fine; the problem is specific to that VNI.

**5. B. Mismatched ESI between the two leaves, or a broken DF election**

Check the type 4 and type 1 state for that segment.

**6. D. The L3VNI, the VRF and the type 5 routes**

Bridging within the L2VNI is clearly fine, so the problem is in routing.

**7. A. A loop, a duplicate address or a misbehaving dual-homed server**

A MAC rapidly reappearing behind different VTEPs is a warning sign; most platforms can alarm on it.

**8. C. The stack is standard BGP and Linux networking, so the same topology can be built in ContainerLab with FRR or SONiC**

Open, standard building blocks make the exact topology reproducible before a change reaches production.

[Back to the questions](13-operating-and-troubleshooting.md#check-your-understanding)
