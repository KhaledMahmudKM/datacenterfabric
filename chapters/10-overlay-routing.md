# Chapter 10: Routing in the Overlay

So far the overlay has only bridged traffic within a single subnet. This chapter adds **routing** between subnets, done inside the fabric itself. The techniques are the distributed anycast gateway, integrated routing and bridging (IRB), and the type 5 route. The combination lets any leaf route for any subnet, right at the edge.

## The default gateway problem

When a host sends to another subnet, it sends to its **default gateway**. In a classic network, the gateway is one router, and traffic to another subnet must travel to wherever that router lives, route there, and travel back. In a fabric with hosts and subnets spread across every leaf, funneling all inter-subnet traffic to one gateway leaf would be a terrible bottleneck and a long, wasteful path.

<figure class="diagram">
<svg viewBox="0 0 680 220" style="width:100%;height:auto;max-width:680px" role="img" aria-label="A centralized gateway forces traffic from a host under Leaf 1 to climb to a single gateway leaf to be routed, then come back, a long path." xmlns="http://www.w3.org/2000/svg">
<style>
.g1{ font-family:var(--sans),sans-serif; }
.g1 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.g1 .gw{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.g1 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.g1 .bad{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
.g1 .s{ fill:var(--text-dim); font-size:13px; }
.g1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="g1">
<rect x="40" y="90" width="90" height="30" rx="6" class="leaf"/><text x="85" y="110" text-anchor="middle" class="s">Leaf 1</text>
<rect x="290" y="40" width="110" height="30" rx="6" class="gw"/><text x="345" y="60" text-anchor="middle" class="s">gateway leaf</text>
<rect x="560" y="90" width="90" height="30" rx="6" class="leaf"/><text x="605" y="110" text-anchor="middle" class="s">Leaf 3</text>
<rect x="30" y="150" width="110" height="28" rx="6" class="node"/><text x="85" y="168" text-anchor="middle" class="s">host, subnet A</text>
<rect x="550" y="150" width="110" height="28" rx="6" class="node"/><text x="605" y="168" text-anchor="middle" class="s">host, subnet B</text>
<line x1="85" y1="120" x2="85" y2="150" stroke="var(--border)" stroke-width="1.4"/><line x1="605" y1="120" x2="605" y2="150" stroke="var(--border)" stroke-width="1.4"/>
<path class="bad" d="M85 90 L345 70"/><path class="bad" d="M345 70 L605 90"/>
<text x="345" y="100" text-anchor="middle" class="lbl">all inter-subnet traffic detours through one gateway</text>
<text x="340" y="204" text-anchor="middle" class="lbl">This is the "traffic trombone": a long path to a single choke point and back.</text>
</g>
</svg>
<figcaption>A centralized gateway creates a trombone: traffic bends up to one gateway leaf to be routed, then back down. It does not scale.</figcaption>
</figure>

## The distributed anycast gateway

EVPN-VXLAN solves this by putting the gateway **everywhere**. Every leaf is configured with the **same gateway IP and the same gateway MAC** for a given subnet. This is the **distributed anycast gateway**. A host's default gateway is always the leaf it is directly attached to, because every leaf answers to that identical gateway address.

<figure class="diagram">
<svg viewBox="0 0 680 220" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Every leaf shares the same gateway IP and MAC for a subnet, so each host routes at its own local leaf with no detour." xmlns="http://www.w3.org/2000/svg">
<style>
.g2{ font-family:var(--sans),sans-serif; }
.g2 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.g2 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.g2 .ok{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.g2 .s{ fill:var(--text-dim); font-size:13px; }
.g2 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.g2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="g2">
<rect x="45" y="70" width="140" height="46" rx="6" class="leaf"/><text x="115" y="90" text-anchor="middle" class="s">Leaf 1</text><text x="115" y="106" text-anchor="middle" class="mono">gw .1 / MAC G</text>
<rect x="275" y="70" width="140" height="46" rx="6" class="leaf"/><text x="345" y="90" text-anchor="middle" class="s">Leaf 2</text><text x="345" y="106" text-anchor="middle" class="mono">gw .1 / MAC G</text>
<rect x="505" y="70" width="140" height="46" rx="6" class="leaf"/><text x="575" y="90" text-anchor="middle" class="s">Leaf 3</text><text x="575" y="106" text-anchor="middle" class="mono">gw .1 / MAC G</text>
<rect x="70" y="150" width="90" height="26" rx="6" class="node"/><text x="115" y="167" text-anchor="middle" class="s">host</text>
<rect x="300" y="150" width="90" height="26" rx="6" class="node"/><text x="345" y="167" text-anchor="middle" class="s">host</text>
<rect x="530" y="150" width="90" height="26" rx="6" class="node"/><text x="575" y="167" text-anchor="middle" class="s">host</text>
<line class="ok" x1="115" y1="150" x2="115" y2="116"/><line class="ok" x1="345" y1="150" x2="345" y2="116"/><line class="ok" x1="575" y1="150" x2="575" y2="116"/>
<text x="345" y="46" text-anchor="middle" class="lbl">same gateway IP and MAC on every leaf</text>
<text x="345" y="200" text-anchor="middle" class="lbl">Each host routes at its own leaf. No detour, and a VM keeps its gateway when it moves.</text>
</g>
</svg>
<figcaption>The distributed anycast gateway. Every leaf is the gateway for the subnet, so routing happens at the ingress leaf. A moving VM never loses its gateway.</figcaption>
</figure>

This has a lovely side effect for VM mobility. Because the gateway IP and MAC are identical on every leaf, a VM that migrates from Leaf 1 to Leaf 3 finds its default gateway already present and unchanged. No gratuitous reconfiguration, no gateway relearning.

## IRB: integrated routing and bridging

A leaf that can both bridge within a subnet and route between subnets is doing **integrated routing and bridging (IRB)**. For routing, the fabric uses two kinds of VNI:

- An **L2VNI** is a bridging segment, one per subnet, exactly what we have used so far.
- An **L3VNI** is a routing segment, one per tenant VRF. It is the transit VNI that carries routed traffic between subnets of the same tenant. A **VRF** (virtual routing and forwarding instance) is the tenant's private routing table.

<figure class="diagram">
<svg viewBox="0 0 680 230" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Two L2VNIs for two subnets both attach to one tenant VRF, which is tied to a single L3VNI used for routed transit between the subnets." xmlns="http://www.w3.org/2000/svg">
<style>
.g3{ font-family:var(--sans),sans-serif; }
.g3 .l2{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.g3 .vrf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.g3 .l3{ fill:var(--panel-hover); stroke:var(--accent); stroke-width:1.5; }
.g3 .ln{ stroke:var(--accent); stroke-width:1.8; fill:none; }
.g3 .s{ fill:var(--text-dim); font-size:13px; }
.g3 .t{ fill:var(--text); font-size:13px; font-weight:700; }
.g3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="g3">
<rect x="40" y="40" width="150" height="44" rx="6" class="l2"/><text x="115" y="60" text-anchor="middle" class="t">L2VNI 10</text><text x="115" y="76" text-anchor="middle" class="s">subnet 10.1.1.0/24</text>
<rect x="40" y="150" width="150" height="44" rx="6" class="l2"/><text x="115" y="170" text-anchor="middle" class="t">L2VNI 20</text><text x="115" y="186" text-anchor="middle" class="s">subnet 10.1.2.0/24</text>
<rect x="290" y="92" width="150" height="50" rx="8" class="vrf"/><text x="365" y="114" text-anchor="middle" class="t">Tenant VRF</text><text x="365" y="130" text-anchor="middle" class="s">private route table</text>
<rect x="540" y="92" width="120" height="50" rx="8" class="l3"/><text x="600" y="114" text-anchor="middle" class="t">L3VNI 999</text><text x="600" y="130" text-anchor="middle" class="s">routed transit</text>
<line class="ln" x1="190" y1="62" x2="290" y2="108" marker-end="url(#ag3)"/><line class="ln" x1="190" y1="172" x2="290" y2="126" marker-end="url(#ag3)"/><line class="ln" x1="440" y1="117" x2="540" y2="117" marker-end="url(#ag3)"/>
<text x="340" y="214" text-anchor="middle" class="lbl">Each subnet has its own L2VNI. All subnets of a tenant share one VRF and one L3VNI for routing.</text>
<defs><marker id="ag3" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>L2VNI, VRF, and L3VNI. Subnets map to L2VNIs for bridging; a tenant's subnets share one VRF and one L3VNI for routing between them.</figcaption>
</figure>

## Symmetric and asymmetric IRB

There are two ways to do IRB, and the difference is worth understanding because fabrics are built around one or the other.

In **asymmetric IRB**, the ingress leaf does all the work: it routes from the source subnet into the destination subnet and then bridges across in the destination's L2VNI. This requires every leaf to be configured with **every** subnet's L2VNI, even subnets it has no local hosts in, so it can bridge into them. That does not scale well.

In **symmetric IRB**, the ingress leaf routes the packet into the shared **L3VNI**, sends it across the fabric in that transit VNI, and the egress leaf routes it out of the L3VNI into the destination subnet. Each leaf only needs the L2VNIs for subnets it actually has, plus the common L3VNI. This scales cleanly and is the method modern fabrics use.

<figure class="diagram">
<svg viewBox="0 0 700 260" style="width:100%;height:auto;max-width:700px" role="img" aria-label="Symmetric IRB: the ingress leaf routes from the source subnet into the L3VNI, the packet crosses the fabric in the L3VNI, and the egress leaf routes it into the destination subnet. Route then route." xmlns="http://www.w3.org/2000/svg">
<style>
.g4{ font-family:var(--sans),sans-serif; }
.g4 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.g4 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.g4 .l3{ fill:var(--panel); stroke:var(--accent); stroke-width:1.5; stroke-dasharray:5 3; }
.g4 .fl{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.g4 .s{ fill:var(--text-dim); font-size:13px; }
.g4 .mono{ fill:var(--text-faint); font-size:12.5px; font-family:var(--mono),monospace; }
.g4 .step{ fill:var(--text); font-size:13px; font-weight:700; }
.g4 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="g4">
<rect x="30" y="100" width="70" height="30" rx="6" class="node"/><text x="65" y="119" text-anchor="middle" class="s">host A</text><text x="65" y="146" text-anchor="middle" class="mono">subnet 10</text>
<rect x="130" y="92" width="100" height="46" rx="6" class="leaf"/><text x="180" y="112" text-anchor="middle" class="s">Leaf 1</text><text x="180" y="128" text-anchor="middle" class="mono">route in</text>
<rect x="290" y="96" width="120" height="40" rx="8" class="l3"/><text x="350" y="120" text-anchor="middle" class="s">L3VNI 999 transit</text>
<rect x="470" y="92" width="100" height="46" rx="6" class="leaf"/><text x="520" y="112" text-anchor="middle" class="s">Leaf 3</text><text x="520" y="128" text-anchor="middle" class="mono">route out</text>
<rect x="600" y="100" width="70" height="30" rx="6" class="node"/><text x="635" y="119" text-anchor="middle" class="s">host B</text><text x="635" y="146" text-anchor="middle" class="mono">subnet 20</text>
<line class="fl" x1="100" y1="115" x2="130" y2="115" marker-end="url(#ag4)"/><line class="fl" x1="230" y1="115" x2="290" y2="115" marker-end="url(#ag4)"/><line class="fl" x1="410" y1="115" x2="470" y2="115" marker-end="url(#ag4)"/><line class="fl" x1="570" y1="115" x2="600" y2="115" marker-end="url(#ag4)"/>
<text x="180" y="78" text-anchor="middle" class="step">route</text><text x="350" y="78" text-anchor="middle" class="step">bridge in L3VNI</text><text x="520" y="78" text-anchor="middle" class="step">route</text>
<text x="350" y="186" text-anchor="middle" class="lbl">Route at ingress, cross in the L3VNI, route at egress. Each leaf needs only its own subnets plus the L3VNI.</text>
<text x="350" y="210" text-anchor="middle" class="lbl">This "route, then route" model is symmetric IRB, the common choice for scalable fabrics.</text>
<defs><marker id="ag4" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Symmetric IRB. Both ends route, and the shared L3VNI carries transit traffic. Leaves need only the subnets they host, which is why this scales.</figcaption>
</figure>

## The type 5 route

How does Leaf 1 know which leaf to send a packet to when the destination is in another subnet? Through the **type 5 route**, the **IP Prefix** advertisement. Where a type 2 route says "I have this host," a type 5 route says "I can reach this IP prefix." It carries the prefix, the tenant VRF's L3VNI, and a **router MAC** extended community so the receiving leaf knows which MAC to use for the inner header on the transit VNI.

<figure class="diagram">
<svg viewBox="0 0 680 210" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Leaf 3 advertises a type 5 route for subnet 20 with its L3VNI and router MAC, and Leaf 1 installs a route to reach subnet 20 via Leaf 3 in the L3VNI." xmlns="http://www.w3.org/2000/svg">
<style>
.g5{ font-family:var(--sans),sans-serif; }
.g5 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.g5 .rr{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.g5 .adv{ stroke:var(--accent); stroke-width:2; fill:none; }
.g5 .s{ fill:var(--text-dim); font-size:13px; }
.g5 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.g5 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="g5">
<rect x="505" y="80" width="150" height="44" rx="6" class="leaf"/><text x="580" y="100" text-anchor="middle" class="s">Leaf 3</text><text x="580" y="116" text-anchor="middle" class="mono">has subnet 20</text>
<rect x="290" y="82" width="100" height="40" rx="6" class="rr"/><text x="340" y="106" text-anchor="middle" class="s">Spine (RR)</text>
<rect x="25" y="80" width="150" height="44" rx="6" class="leaf"/><text x="100" y="100" text-anchor="middle" class="s">Leaf 1</text><text x="100" y="116" text-anchor="middle" class="mono">installs route</text>
<line class="adv" x1="505" y1="102" x2="390" y2="102" marker-end="url(#ag5)"/><line class="adv" x1="290" y1="102" x2="175" y2="102" marker-end="url(#ag5)"/>
<text x="340" y="66" text-anchor="middle" class="mono">type 5: 10.1.2.0/24, L3VNI 999, router-MAC of Leaf 3</text>
<text x="40" y="152" class="mono">reach 10.1.2.0/24 via Leaf 3 in L3VNI 999</text>
<text x="340" y="190" text-anchor="middle" class="lbl">Type 5 carries subnets, not hosts. It is how the overlay routes between subnets and to external networks.</text>
<defs><marker id="ag5" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>The type 5 route advertises a prefix with its L3VNI and router MAC. This is also how the fabric reaches external networks and summarized routes.</figcaption>
</figure>

Type 5 routes are also the bridge to the outside world. A **border leaf** that connects the fabric to the WAN or to external routers advertises external destinations into the fabric as type 5 routes, and advertises the fabric's internal subnets outward, so tenants can reach and be reached beyond the data center.

## Configuration shape

The tenant routing config adds a VRF tied to the L3VNI, and tells BGP to advertise the VRF's connected subnets as type 5. In **FRR** it looks like this:

```
vrf TENANT_A
 vni 999
exit-vrf
!
router bgp 65101 vrf TENANT_A
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 address-family l2vpn evpn
  advertise ipv4 unicast        # export the VRF's subnets as type 5
 exit-address-family
```

The anycast gateway itself is configured per subnet by giving the subnet's switched virtual interface the shared gateway IP and MAC on every leaf. On SONiC the same structure exists through its VRF and VLAN-interface configuration, mapping the VRF to an L3VNI and enabling type 5 advertisement.

## Where we are

The fabric can now bridge within a subnet and route between subnets, all at the edge, with a gateway that follows VMs wherever they go. One assumption has quietly held throughout: each host connects to exactly one leaf. Real deployments dual-home servers to two leaves for redundancy, and that raises genuinely tricky questions about loops and duplicate delivery. Those are the multihoming problems, and the route types 1 and 4 we have set aside, which the next chapter finally takes up.
