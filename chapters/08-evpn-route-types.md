# Chapter 8: EVPN Route Types

EVPN carries its information as **routes**, and each route has a **type** that says what kind of information it holds. There are several defined types, but five do almost all the work in a data center fabric. This chapter introduces all five, then looks closely at the two you will see most, types 2 and 3.

## The five route types at a glance

<figure class="diagram">
<svg viewBox="0 0 700 340" style="width:100%;height:auto;max-width:700px" role="img" aria-label="A table of the five EVPN route types: type 1 Ethernet auto-discovery, type 2 MAC/IP advertisement, type 3 inclusive multicast, type 4 Ethernet segment, type 5 IP prefix, each with its purpose." xmlns="http://www.w3.org/2000/svg">
<style>
.r1{ font-family:var(--sans),sans-serif; }
.r1 .row{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.2; }
.r1 .rowalt{ fill:var(--panel); stroke:var(--border); stroke-width:1.2; }
.r1 .hdr{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.4; }
.r1 .num{ fill:var(--accent); font-size:18px; font-weight:700; }
.r1 .t{ fill:var(--text); font-size:13px; font-weight:700; }
.r1 .s{ fill:var(--text-dim); font-size:13px; }
.r1 .h{ fill:var(--text); font-size:15px; font-weight:700; }
</style>
<g class="r1">
<rect x="20" y="20" width="660" height="34" rx="6" class="hdr"/>
<text x="55" y="42" text-anchor="middle" class="h">Type</text><text x="210" y="42" text-anchor="middle" class="h">Name</text><text x="480" y="42" text-anchor="middle" class="h">What it is for</text>
<rect x="20" y="58" width="660" height="52" rx="6" class="row"/>
<text x="55" y="90" text-anchor="middle" class="num">1</text><text x="130" y="81" class="t">Ethernet</text><text x="130" y="97" class="t">Auto-Discovery</text><text x="330" y="81" class="s">Multihoming support: fast failover and</text><text x="330" y="97" class="s">aliasing across links to a dual-homed host.</text>
<rect x="20" y="114" width="660" height="52" rx="6" class="rowalt"/>
<text x="55" y="146" text-anchor="middle" class="num">2</text><text x="130" y="137" class="t">MAC/IP</text><text x="130" y="153" class="t">Advertisement</text><text x="330" y="137" class="s">The workhorse: advertises a host MAC and</text><text x="330" y="153" class="s">optionally its IP. This is host learning.</text>
<rect x="20" y="170" width="660" height="52" rx="6" class="row"/>
<text x="55" y="202" text-anchor="middle" class="num">3</text><text x="130" y="193" class="t">Inclusive</text><text x="130" y="209" class="t">Multicast (IMET)</text><text x="330" y="193" class="s">VTEP discovery: who is in this VNI, so BUM</text><text x="330" y="209" class="s">traffic can be replicated to them.</text>
<rect x="20" y="226" width="660" height="52" rx="6" class="rowalt"/>
<text x="55" y="258" text-anchor="middle" class="num">4</text><text x="130" y="249" class="t">Ethernet</text><text x="130" y="265" class="t">Segment</text><text x="330" y="249" class="s">Multihoming support: discover VTEPs sharing</text><text x="330" y="265" class="s">a link bundle and elect a designated forwarder.</text>
<rect x="20" y="282" width="660" height="52" rx="6" class="row"/>
<text x="55" y="314" text-anchor="middle" class="num">5</text><text x="130" y="305" class="t">IP Prefix</text><text x="330" y="305" class="s">Advertises an IP prefix (a subnet or summary)</text><text x="330" y="321" class="s">for routing between subnets in the overlay.</text>
</g>
</svg>
<figcaption>The five route types. Types 2, 3, and 5 carry everyday forwarding; types 1 and 4 support multihoming, covered in Chapter 11.</figcaption>
</figure>

A good way to group them: **type 2** advertises individual hosts, **type 3** discovers which VTEPs belong to a segment, **type 5** advertises whole subnets for inter-subnet routing, and **types 1 and 4** exist to make dual-homed hosts work. We focus on 2 and 3 now, return to 5 in Chapter 10, and cover 1 and 4 in Chapter 11.

## Type 2: the MAC/IP advertisement

The type 2 route is the one you will read in `show` output more than any other. It announces: "I, this VTEP, have this host." It can carry just a MAC address, or a MAC paired with an IP address.

<figure class="diagram">
<svg viewBox="0 0 700 290" style="width:100%;height:auto;max-width:700px" role="img" aria-label="Anatomy of a type 2 route showing the route distinguisher, MAC address, IP address, the L2VNI label, an optional L3VNI label, and the route target community." xmlns="http://www.w3.org/2000/svg">
<style>
.r2{ font-family:var(--sans),sans-serif; }
.r2 .field{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.3; }
.r2 .key{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.r2 .k{ fill:var(--text); font-size:13px; font-weight:700; }
.r2 .v{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.r2 .d{ fill:var(--text-dim); font-size:13px; }
.r2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.r2 .title{ fill:var(--text); font-size:15px; font-weight:700; }
</style>
<g class="r2">
<text x="350" y="26" text-anchor="middle" class="title">Type 2: MAC/IP Advertisement Route</text>
<rect x="40" y="44" width="620" height="34" rx="5" class="field"/><text x="56" y="66" class="k">Route Distinguisher</text><text x="644" y="66" text-anchor="end" class="v">10.0.1.1:10</text>
<rect x="40" y="84" width="620" height="34" rx="5" class="key"/><text x="56" y="106" class="k">MAC address</text><text x="644" y="106" text-anchor="end" class="v">00:11:22:aa:bb:cc</text>
<rect x="40" y="124" width="620" height="34" rx="5" class="field"/><text x="56" y="146" class="k">IP address (optional)</text><text x="644" y="146" text-anchor="end" class="v">10.1.1.5</text>
<rect x="40" y="164" width="620" height="34" rx="5" class="field"/><text x="56" y="186" class="k">MPLS Label 1  =  L2VNI</text><text x="644" y="186" text-anchor="end" class="v">10010</text>
<rect x="40" y="204" width="620" height="34" rx="5" class="field"/><text x="56" y="226" class="k">MPLS Label 2  =  L3VNI (optional)</text><text x="644" y="226" text-anchor="end" class="v">10999</text>
<rect x="40" y="244" width="620" height="34" rx="5" class="field"/><text x="56" y="266" class="k">Extended communities</text><text x="644" y="266" text-anchor="end" class="v">RT 65000:10, Router-MAC</text>
<text x="350" y="40" text-anchor="middle" class="lbl"></text>
</g>
</svg>
<figcaption>The anatomy of a type 2 route. The MAC is always present; the IP, the second label, and the router MAC appear when the route also supports routing (Chapter 10).</figcaption>
</figure>

Two things in that structure deserve attention now:

- The **IP field** is what enables ARP suppression. If the route carries both MAC and IP, every remote VTEP learns the IP-to-MAC binding for that host. When a local server later ARPs for that IP, the VTEP already knows the answer and replies locally, so the ARP broadcast never crosses the fabric.
- The **VNI labels** map the host into a segment. Label 1 is the **L2VNI**, the bridging segment the host sits in. The optional Label 2 is the **L3VNI**, used for routing between segments, which Chapter 10 explains. A type 2 route carrying both a MAC and an IP, and both labels, is the heart of integrated bridging and routing.

## Type 3: inclusive multicast, the VTEP roll call

The type 2 route handles known unicast beautifully, but BUM traffic from Chapter 6 still has to reach every VTEP in a segment. The type 3 route, **Inclusive Multicast Ethernet Tag (IMET)**, solves the discovery half of that problem. Each VTEP that has at least one member in a given VNI advertises a type 3 route for that VNI, in effect announcing "I am part of VNI 10, send BUM for it to my VTEP IP."

<figure class="diagram">
<svg viewBox="0 0 700 250" style="width:100%;height:auto;max-width:700px" role="img" aria-label="Each VTEP in a VNI sends a type 3 route announcing its membership, so every VTEP builds a list of peers to replicate BUM traffic to." xmlns="http://www.w3.org/2000/svg">
<style>
.r3{ font-family:var(--sans),sans-serif; }
.r3 .vtep{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.r3 .rr{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.r3 .ln{ stroke:var(--accent); stroke-width:1.8; fill:none; }
.r3 .s{ fill:var(--text-dim); font-size:13px; }
.r3 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.r3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.r3 .tbl{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.3; }
.r3 .t{ fill:var(--text); font-size:13px; font-weight:700; }
</style>
<g class="r3">
<rect x="285" y="24" width="130" height="34" rx="6" class="rr"/><text x="350" y="46" text-anchor="middle" class="s">route reflector</text>
<rect x="30" y="92" width="86" height="32" rx="6" class="vtep"/><text x="73" y="112" text-anchor="middle" class="s">VTEP 1</text>
<rect x="180" y="92" width="86" height="32" rx="6" class="vtep"/><text x="223" y="112" text-anchor="middle" class="s">VTEP 2</text>
<rect x="430" y="92" width="86" height="32" rx="6" class="vtep"/><text x="473" y="112" text-anchor="middle" class="s">VTEP 3</text>
<line class="ln" x1="73" y1="92" x2="320" y2="52" marker-end="url(#ar3)"/><line class="ln" x1="223" y1="92" x2="335" y2="58" marker-end="url(#ar3)"/><line class="ln" x1="473" y1="92" x2="380" y2="56" marker-end="url(#ar3)"/>
<text x="436" y="46" class="mono">type 3: VNI 10, me</text>
<rect x="520" y="140" width="160" height="92" rx="8" class="tbl"/><text x="600" y="160" text-anchor="middle" class="t">VNI 10 flood list</text><text x="600" y="182" text-anchor="middle" class="mono">VTEP 1 loopback</text><text x="600" y="200" text-anchor="middle" class="mono">VTEP 2 loopback</text><text x="600" y="218" text-anchor="middle" class="mono">VTEP 3 loopback</text>
<text x="30" y="170" class="lbl">Each VTEP collects the others' type 3 routes</text><text x="30" y="188" class="lbl">and builds a flood list for the VNI. BUM is then</text><text x="30" y="206" class="lbl">replicated (head-end) to exactly those peers.</text>
<defs><marker id="ar3" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Type 3 routes let each VTEP build an exact flood list per VNI. BUM replication now targets only the VTEPs that actually have members, learned from the control plane.</figcaption>
</figure>

This replaces the manual or multicast-based BUM configuration of plain flood-and-learn. The flood list is now built automatically from BGP, it includes exactly the VTEPs that have members, and it updates itself as VTEPs join or leave the VNI. Most EVPN-VXLAN fabrics use this with head-end replication and never touch underlay multicast at all.

## How the types work together

It helps to see the division of labor in one picture. Known unicast rides type 2. BUM discovery rides type 3. Routing between subnets rides type 5. Multihoming rides types 1 and 4.

<figure class="diagram">
<svg viewBox="0 0 700 210" style="width:100%;height:auto;max-width:700px" role="img" aria-label="A mapping from forwarding jobs to the route types that serve them: unicast uses type 2, BUM uses type 3, inter-subnet routing uses type 5, multihoming uses types 1 and 4." xmlns="http://www.w3.org/2000/svg">
<style>
.r4{ font-family:var(--sans),sans-serif; }
.r4 .job{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.r4 .rt{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.r4 .ln{ stroke:var(--accent); stroke-width:1.8; fill:none; }
.r4 .s{ fill:var(--text-dim); font-size:13px; }
.r4 .t{ fill:var(--text); font-size:13px; font-weight:700; }
</style>
<g class="r4">
<rect x="40" y="30" width="200" height="30" rx="6" class="job"/><text x="140" y="50" text-anchor="middle" class="s">reach a known host</text>
<rect x="40" y="76" width="200" height="30" rx="6" class="job"/><text x="140" y="96" text-anchor="middle" class="s">flood BUM in a VNI</text>
<rect x="40" y="122" width="200" height="30" rx="6" class="job"/><text x="140" y="142" text-anchor="middle" class="s">route between subnets</text>
<rect x="40" y="168" width="200" height="30" rx="6" class="job"/><text x="140" y="188" text-anchor="middle" class="s">dual-homed host</text>
<rect x="460" y="30" width="200" height="30" rx="6" class="rt"/><text x="560" y="50" text-anchor="middle" class="t">Type 2</text>
<rect x="460" y="76" width="200" height="30" rx="6" class="rt"/><text x="560" y="96" text-anchor="middle" class="t">Type 3</text>
<rect x="460" y="122" width="200" height="30" rx="6" class="rt"/><text x="560" y="142" text-anchor="middle" class="t">Type 5</text>
<rect x="460" y="168" width="200" height="30" rx="6" class="rt"/><text x="560" y="188" text-anchor="middle" class="t">Types 1 + 4</text>
<line class="ln" x1="240" y1="45" x2="460" y2="45"/><line class="ln" x1="240" y1="91" x2="460" y2="91"/><line class="ln" x1="240" y1="137" x2="460" y2="137"/><line class="ln" x1="240" y1="183" x2="460" y2="183"/>
</g>
</svg>
<figcaption>Each forwarding job maps to a route type. Keep this picture in mind as the next chapters walk through each job in detail.</figcaption>
</figure>

The next chapter puts types 2 and 3 to work together, walking a real packet exchange through an EVPN-VXLAN fabric and showing ARP suppression in action.
