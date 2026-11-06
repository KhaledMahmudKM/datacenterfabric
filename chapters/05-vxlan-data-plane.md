# Chapter 5: The VXLAN Data Plane

This chapter is about the **data plane**: the exact shape of a VXLAN packet and what happens to a frame as it crosses the fabric. VXLAN is defined in **RFC 7348**. The term "data plane" means the machinery that actually forwards packets, as opposed to the "control plane" that decides where they should go. We cover the control plane in Part III. For now, assume each switch somehow already knows where to send things, and watch the packets move.

## The VTEP

The device that adds and removes VXLAN headers is a **VXLAN Tunnel Endpoint**, or **VTEP**. In a spine-leaf fabric, the leaves are the VTEPs. A VTEP has one foot in each world: on the tenant side it sees ordinary Ethernet frames from servers, and on the fabric side it sends and receives VXLAN-encapsulated IP packets.

<figure class="diagram">
<svg viewBox="0 0 680 220" style="width:100%;height:auto;max-width:680px" role="img" aria-label="A VTEP sits between the tenant-facing side where it handles Ethernet frames and the fabric-facing side where it handles VXLAN over IP." xmlns="http://www.w3.org/2000/svg">
<style>
.v1{ font-family:var(--sans),sans-serif; }
.v1 .vtep{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.8; }
.v1 .srv{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.v1 .fab{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.v1 .ln{ stroke:var(--border); stroke-width:1.4; fill:none; }
.v1 .s{ fill:var(--text-dim); font-size:13px; }
.v1 .h{ fill:var(--text); font-size:15px; font-weight:700; }
.v1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.v1 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
</style>
<g class="v1">
<rect x="270" y="78" width="140" height="64" rx="8" class="vtep"/><text x="340" y="104" text-anchor="middle" class="h">VTEP</text><text x="340" y="122" text-anchor="middle" class="s">(the leaf)</text>
<text x="340" y="162" text-anchor="middle" class="mono">VTEP IP = loopback</text>
<rect x="40" y="88" width="80" height="44" rx="6" class="srv"/><text x="80" y="108" text-anchor="middle" class="s">server</text><text x="80" y="124" text-anchor="middle" class="lbl">tenant</text>
<line class="ln" x1="120" y1="110" x2="270" y2="110"/>
<text x="195" y="100" text-anchor="middle" class="lbl">Ethernet frames</text>
<rect x="560" y="88" width="90" height="44" rx="6" class="fab"/><text x="605" y="108" text-anchor="middle" class="s">fabric</text><text x="605" y="124" text-anchor="middle" class="lbl">spines</text>
<line class="ln" x1="410" y1="110" x2="560" y2="110"/>
<text x="485" y="100" text-anchor="middle" class="lbl">VXLAN over IP</text>
<text x="340" y="200" text-anchor="middle" class="s">In: plain frames from servers.  Out: those frames wrapped as VXLAN and routed over the underlay.</text>
</g>
</svg>
<figcaption>A VTEP translates between the tenant Ethernet world and the VXLAN-over-IP fabric. Its address is its underlay loopback.</figcaption>
</figure>

A VTEP's address is its **underlay loopback** from Chapter 3. That is the tie between the two layers: overlay tunnels are built between loopback addresses, and the underlay already knows how to reach every loopback over ECMP.

## The packet, layer by layer

When a VTEP encapsulates a tenant frame, it prepends four headers to it. From outside in, the finished packet on the fabric looks like this:

<figure class="diagram">
<svg viewBox="0 0 700 250" style="width:100%;height:auto;max-width:700px" role="img" aria-label="A VXLAN packet shown as nested headers: outer Ethernet, outer IP, outer UDP with destination port 4789, the 8-byte VXLAN header carrying the VNI, then the original inner Ethernet frame and payload." xmlns="http://www.w3.org/2000/svg">
<style>
.v2{ font-family:var(--sans),sans-serif; }
.v2 .oeth{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.v2 .oip{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.4; }
.v2 .oudp{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.v2 .vx{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.8; }
.v2 .inner{ fill:var(--panel); stroke:var(--text-faint); stroke-width:1.4; stroke-dasharray:4 3; }
.v2 .t{ fill:var(--text); font-size:13px; font-weight:700; }
.v2 .s{ fill:var(--text-dim); font-size:13px; }
.v2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.v2 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; font-weight:700; }
</style>
<g class="v2">
<rect x="20" y="40" width="124" height="70" rx="6" class="oeth"/><text x="82" y="68" text-anchor="middle" class="t">Outer</text><text x="82" y="84" text-anchor="middle" class="t">Ethernet</text><text x="82" y="100" text-anchor="middle" class="lbl">leaf to leaf MAC</text>
<rect x="148" y="40" width="124" height="70" rx="6" class="oip"/><text x="210" y="68" text-anchor="middle" class="t">Outer IP</text><text x="210" y="86" text-anchor="middle" class="s">src/dst =</text><text x="210" y="100" text-anchor="middle" class="lbl">VTEP loopbacks</text>
<rect x="276" y="40" width="124" height="70" rx="6" class="oudp"/><text x="338" y="68" text-anchor="middle" class="t">Outer UDP</text><text x="338" y="86" text-anchor="middle" class="mono">dst 4789</text><text x="338" y="100" text-anchor="middle" class="lbl">src = flow hash</text>
<rect x="404" y="40" width="112" height="70" rx="6" class="vx"/><text x="460" y="64" text-anchor="middle" class="t">VXLAN</text><text x="460" y="80" text-anchor="middle" class="s">8 bytes</text><text x="460" y="98" text-anchor="middle" class="mono">24-bit VNI</text>
<rect x="520" y="40" width="160" height="70" rx="6" class="inner"/><text x="600" y="64" text-anchor="middle" class="t">Inner Ethernet frame</text><text x="600" y="82" text-anchor="middle" class="s">tenant MACs and IPs,</text><text x="600" y="98" text-anchor="middle" class="s">and the payload</text>
<text x="210" y="150" text-anchor="middle" class="lbl">added by the VTEP (the outer wrap)</text>
<line x1="20" y1="124" x2="400" y2="124" stroke="var(--text-faint)" stroke-width="1"/>
<text x="600" y="150" text-anchor="middle" class="lbl">the original frame, untouched</text>
<line x1="520" y1="124" x2="680" y2="124" stroke="var(--text-faint)" stroke-width="1"/>
<text x="350" y="196" text-anchor="middle" class="s">This is "MAC-in-UDP": a whole Ethernet frame carried as the payload of a UDP datagram.</text>
<text x="350" y="222" text-anchor="middle" class="lbl">VXLAN adds roughly 50 bytes of overhead, which is why fabrics raise the MTU.</text>
</g>
</svg>
<figcaption>The VXLAN packet. Four outer headers wrap the original frame. Only the VXLAN header is new to you; the rest are ordinary Ethernet, IP, and UDP.</figcaption>
</figure>

Walking the headers from outside in:

- **Outer Ethernet.** Normal MAC addressing for the next hop across the fabric, rewritten at each router hop like any routed packet.
- **Outer IP.** The source is the sending VTEP's loopback; the destination is the receiving VTEP's loopback. This is what the underlay routes on.
- **Outer UDP.** The destination port is the well-known VXLAN port **4789**. The source port is not a real port at all; the VTEP fills it with a hash of the inner flow. This is a deliberate trick, explained below.
- **VXLAN header.** Eight bytes. Its important content is the 24-bit **VXLAN Network Identifier (VNI)**, plus a flag bit that marks the VNI as valid.

Then comes the inner Ethernet frame exactly as the server sent it.

## The VNI and the end of the VLAN ceiling

The **VNI** is the overlay's segment number. It plays the role that a VLAN ID played inside a single switch, but where a VLAN ID is 12 bits and tops out at 4094 usable values, the VNI is **24 bits**, giving about **16 million** segments. Each VNI is an isolated Layer 2 (or, later, Layer 3) segment. Two tenants can both use VLAN 10 locally on their own servers, and the fabric keeps them apart by mapping each to a different VNI.

<figure class="diagram">
<svg viewBox="0 0 640 170" style="width:100%;height:auto;max-width:640px" role="img" aria-label="A 12-bit VLAN id allows about four thousand segments while a 24-bit VNI allows about sixteen million." xmlns="http://www.w3.org/2000/svg">
<style>
.v3{ font-family:var(--sans),sans-serif; }
.v3 .small{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.v3 .big{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.8; }
.v3 .h{ fill:var(--text); font-size:15px; font-weight:700; }
.v3 .s{ fill:var(--text-dim); font-size:13px; }
.v3 .num{ fill:var(--accent); font-size:20px; font-weight:700; }
.v3 .numd{ fill:var(--text-dim); font-size:20px; font-weight:700; }
.v3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="v3">
<rect x="40" y="30" width="240" height="90" rx="8" class="small"/><text x="160" y="56" text-anchor="middle" class="h">VLAN ID</text><text x="160" y="76" text-anchor="middle" class="s">12 bits</text><text x="160" y="106" text-anchor="middle" class="numd">4,094</text>
<rect x="360" y="30" width="240" height="90" rx="8" class="big"/><text x="480" y="56" text-anchor="middle" class="h">VXLAN VNI</text><text x="480" y="76" text-anchor="middle" class="s">24 bits</text><text x="480" y="106" text-anchor="middle" class="num">~16,000,000</text>
<text x="320" y="80" text-anchor="middle" class="h">to</text>
<text x="320" y="150" text-anchor="middle" class="lbl">Same role, far larger range. The VNI is why VXLAN scales to large multi-tenant clouds.</text>
</g>
</svg>
<figcaption>The VNI is a 24-bit segment identifier. It removes the 4094-VLAN limit that constrained the old Layer 2 designs.</figcaption>
</figure>

## Why UDP, and the source-port trick

Carrying Ethernet inside UDP looks odd at first. The reason is the fabric's ECMP from Part I. Switches spread traffic across equal paths by hashing fields from each packet, typically the source and destination IP and the source and destination UDP or TCP ports. If every VXLAN packet between two VTEPs had identical outer headers, that hash would be constant, and all the tunneled traffic would pin to a single spine, wasting the others.

VXLAN avoids this by setting the **outer UDP source port to a hash of the inner flow**. Different inner flows, meaning different tenant conversations, produce different outer source ports, so the fabric hashes them onto different spines. The overlay traffic spreads across all the ECMP paths just like native traffic, even though to the spines it is all just UDP between two loopbacks.

<figure class="diagram">
<svg viewBox="0 0 680 200" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Three different inner flows get three different outer UDP source ports, so the fabric hashes them onto three different spines." xmlns="http://www.w3.org/2000/svg">
<style>
.v4{ font-family:var(--sans),sans-serif; }
.v4 .vtep{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.v4 .spine{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.v4 .f1{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.v4 .s{ fill:var(--text-dim); font-size:13px; }
.v4 .mono{ fill:var(--text-faint); font-size:12.5px; font-family:var(--mono),monospace; }
.v4 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="v4">
<rect x="30" y="24" width="90" height="150" rx="6" class="vtep"/><text x="75" y="104" text-anchor="middle" class="s">VTEP</text>
<rect x="300" y="24" width="90" height="26" rx="6" class="spine"/><text x="345" y="42" text-anchor="middle" class="s">Spine 1</text>
<rect x="300" y="86" width="90" height="26" rx="6" class="spine"/><text x="345" y="104" text-anchor="middle" class="s">Spine 2</text>
<rect x="300" y="148" width="90" height="26" rx="6" class="spine"/><text x="345" y="166" text-anchor="middle" class="s">Spine 3</text>
<line class="f1" x1="120" y1="37" x2="300" y2="37"/><line class="f1" x1="120" y1="99" x2="300" y2="99"/><line class="f1" x1="120" y1="161" x2="300" y2="161"/>
<text x="210" y="29" text-anchor="middle" class="mono">sport 51001</text><text x="210" y="91" text-anchor="middle" class="mono">sport 49210</text><text x="210" y="153" text-anchor="middle" class="mono">sport 60733</text>
<rect x="440" y="24" width="90" height="150" rx="6" class="vtep"/><text x="485" y="104" text-anchor="middle" class="s">VTEP</text>
<line class="f1" x1="390" y1="37" x2="440" y2="37"/><line class="f1" x1="390" y1="99" x2="440" y2="99"/><line class="f1" x1="390" y1="161" x2="440" y2="161"/>
<text x="340" y="196" text-anchor="middle" class="lbl">Each inner flow hashes to a different outer source port, so the fabric uses every spine.</text>
</g>
</svg>
<figcaption>The source-port trick. Encoding the inner flow into the outer UDP source port lets the underlay load-balance tunneled traffic across all spines.</figcaption>
</figure>

## A frame's round trip

Putting it together, here is the life of one frame from VM A under Leaf 1 to VM B under Leaf 40, both in the same VNI:

1. VM A sends an ordinary Ethernet frame toward VM B's MAC address.
2. Leaf 1, the ingress VTEP, looks up where VM B's MAC lives and finds it behind Leaf 40's VTEP. It wraps the frame: VXLAN header with the segment's VNI, UDP with port 4789 and a flow-hashed source port, outer IP from Leaf 1's loopback to Leaf 40's loopback.
3. The fabric routes the outer IP packet, spreading it across spines by ECMP. The spines see only UDP between two loopbacks and never look inside.
4. Leaf 40, the egress VTEP, receives the packet, strips the outer headers, reads the VNI to know which segment the frame belongs to, and delivers the original frame to VM B.

<figure class="diagram">
<svg viewBox="0 0 700 200" style="width:100%;height:auto;max-width:700px" role="img" aria-label="VM A frame is encapsulated by Leaf 1, routed through a spine, and decapsulated by Leaf 40 to reach VM B." xmlns="http://www.w3.org/2000/svg">
<style>
.v5{ font-family:var(--sans),sans-serif; }
.v5 .node{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.v5 .vtep{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.v5 .fl{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.v5 .s{ fill:var(--text-dim); font-size:13px; }
.v5 .step{ fill:var(--accent); font-size:13px; font-weight:700; }
.v5 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="v5">
<rect x="20" y="86" width="70" height="30" rx="6" class="node"/><text x="55" y="105" text-anchor="middle" class="s">VM A</text>
<rect x="130" y="80" width="90" height="42" rx="6" class="vtep"/><text x="175" y="99" text-anchor="middle" class="s">Leaf 1</text><text x="175" y="114" text-anchor="middle" class="lbl">encap</text>
<rect x="320" y="84" width="80" height="34" rx="6" class="node"/><text x="360" y="105" text-anchor="middle" class="s">Spine</text>
<rect x="490" y="80" width="90" height="42" rx="6" class="vtep"/><text x="535" y="99" text-anchor="middle" class="s">Leaf 40</text><text x="535" y="114" text-anchor="middle" class="lbl">decap</text>
<rect x="620" y="86" width="60" height="30" rx="6" class="node"/><text x="650" y="105" text-anchor="middle" class="s">VM B</text>
<line class="fl" x1="90" y1="101" x2="130" y2="101" marker-end="url(#av5)"/><line class="fl" x1="220" y1="101" x2="320" y2="101" marker-end="url(#av5)"/><line class="fl" x1="400" y1="101" x2="490" y2="101" marker-end="url(#av5)"/><line class="fl" x1="580" y1="101" x2="620" y2="101" marker-end="url(#av5)"/>
<text x="55" y="150" text-anchor="middle" class="step">1</text><text x="175" y="150" text-anchor="middle" class="step">2 wrap</text><text x="360" y="150" text-anchor="middle" class="step">3 route</text><text x="535" y="150" text-anchor="middle" class="step">4 unwrap</text><text x="650" y="150" text-anchor="middle" class="step">deliver</text>
<text x="350" y="182" text-anchor="middle" class="lbl">The spine never sees VM A or VM B. It routes UDP between Leaf 1 and Leaf 40 loopbacks.</text>
<defs><marker id="av5" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>One frame, end to end. Encapsulate at ingress, route across the fabric, decapsulate at egress.</figcaption>
</figure>

## The open question

Step 2 hid something important. Leaf 1 had to **know** that VM B's MAC address lives behind Leaf 40. How does it learn that? And what does a VTEP do with a broadcast frame, which by definition has no single destination VTEP?

The original VXLAN specification answered this with flooding and data-plane learning, a method that works but scales poorly. The next chapter examines that approach and its limits, which sets up the reason EVPN exists.
