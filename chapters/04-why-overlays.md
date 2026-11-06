# Chapter 4: Why Overlays

The routed underlay from Chapter 3 is fast, redundant, and loop-free. It also cannot, on its own, give applications what many of them still demand. This chapter explains that gap and introduces the idea that fills it: the **overlay**.

## The conflict

The fabric is routed everywhere. That is exactly why it scales. But workloads frequently need **Layer 2 adjacency**, meaning two servers that behave as though they sit on the same Ethernet segment, in the same subnet, able to broadcast to each other.

Why would they need that in a modern design? Several reasons persist:

- **Virtual machine mobility.** When a VM moves from a host under Leaf 1 to a host under Leaf 40, it expects to keep its IP address and its default gateway. That only works if both hosts appear to be on the same Layer 2 segment.
- **Clustering and legacy software.** Many clustering, failover, and discovery protocols assume a shared broadcast domain.
- **Multi-tenancy.** A cloud platform must give each tenant the illusion of their own private Layer 2 network, isolated from every other tenant, regardless of which physical leaf their servers land on.

<figure class="diagram">
<svg viewBox="0 0 680 270" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Two servers on the same subnet sit under different leaves separated by a routed fabric, which normally blocks their Layer 2 adjacency." xmlns="http://www.w3.org/2000/svg">
<style>
.o1{ font-family:var(--sans),sans-serif; }
.o1 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.o1 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.o1 .srv{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.4; }
.o1 .ln{ stroke:var(--border); stroke-width:1.4; fill:none; }
.o1 .bar{ stroke:var(--text-faint); stroke-width:2; fill:none; stroke-dasharray:2 3; }
.o1 .s{ fill:var(--text-dim); font-size:13px; }
.o1 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.o1 .warn{ fill:var(--text-faint); font-size:13px; }
</style>
<g class="o1">
<line class="ln" x1="340" y1="48" x2="130" y2="120"/><line class="ln" x1="340" y1="48" x2="550" y2="120"/>
<rect x="290" y="24" width="100" height="26" rx="6" class="spine"/><text x="340" y="42" text-anchor="middle" class="s">Spine</text>
<rect x="80" y="120" width="100" height="26" rx="6" class="leaf"/><text x="130" y="138" text-anchor="middle" class="s">Leaf 1</text>
<rect x="500" y="120" width="100" height="26" rx="6" class="leaf"/><text x="550" y="138" text-anchor="middle" class="s">Leaf 40</text>
<line class="ln" x1="130" y1="146" x2="130" y2="174"/><line class="ln" x1="550" y1="146" x2="550" y2="174"/>
<rect x="75" y="174" width="110" height="44" rx="6" class="srv"/><text x="130" y="193" text-anchor="middle" class="s">VM A</text><text x="130" y="211" text-anchor="middle" class="mono">10.1.1.5/24</text>
<rect x="495" y="174" width="110" height="44" rx="6" class="srv"/><text x="550" y="193" text-anchor="middle" class="s">VM B</text><text x="550" y="211" text-anchor="middle" class="mono">10.1.1.6/24</text>
<line class="bar" x1="340" y1="70" x2="340" y2="222"/>
<rect x="268" y="134" width="144" height="58" fill="var(--bg)" stroke="none"/>
<text x="340" y="150" text-anchor="middle" class="warn">routed fabric</text><text x="340" y="167" text-anchor="middle" class="warn">breaks the</text><text x="340" y="184" text-anchor="middle" class="warn">Layer 2 segment</text>
<text x="340" y="252" text-anchor="middle" class="s">Same subnet, different leaves. The routed underlay will not bridge them by itself.</text>
</g>
</svg>
<figcaption>The conflict. Two VMs in one subnet land under different leaves, but the fabric between them routes rather than bridges.</figcaption>
</figure>

We could solve this the old way, by stretching VLANs across the whole fabric at Layer 2. But that brings back spanning tree, broadcast storms, and the 4094-VLAN ceiling, which are exactly the problems we escaped in Part I. We do not want to give those up.

## The overlay idea

The solution is to keep the fabric routed and build a second, virtual network on top of it. This upper network is the **overlay**. The physical routed fabric underneath is the **underlay**, the name we have been using all along.

<figure class="diagram">
<svg viewBox="0 0 660 288" style="width:100%;height:auto;max-width:660px" role="img" aria-label="Two stacked planes: a lower physical underlay of routed switches, and an upper overlay where virtual Layer 2 segments connect servers directly." xmlns="http://www.w3.org/2000/svg">
<style>
.o2{ font-family:var(--sans),sans-serif; }
.o2 .plane{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.o2 .overlay{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.o2 .node{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.o2 .ov{ stroke:var(--accent); stroke-width:2.4; fill:none; }
.o2 .ul{ stroke:var(--text-faint); stroke-width:1.4; fill:none; }
.o2 .conn{ stroke:var(--accent); stroke-width:1.4; stroke-dasharray:4 4; fill:none; }
.o2 .h{ fill:var(--text); font-size:15px; font-weight:700; }
.o2 .s{ fill:var(--text-dim); font-size:13px; }
.o2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="o2">
<rect x="40" y="20" width="580" height="96" rx="8" class="overlay" opacity="0.25"/>
<text x="60" y="40" class="h">Overlay</text><text x="60" y="56" class="lbl">virtual Layer 2 segments</text>
<rect x="210" y="64" width="80" height="30" rx="6" class="node"/><text x="250" y="83" text-anchor="middle" class="s">VM A</text>
<rect x="430" y="64" width="80" height="30" rx="6" class="node"/><text x="470" y="83" text-anchor="middle" class="s">VM B</text>
<line class="ov" x1="290" y1="79" x2="430" y2="79"/>
<text x="360" y="72" text-anchor="middle" class="lbl">looks like one wire</text>
<rect x="40" y="164" width="580" height="96" rx="8" class="plane"/>
<text x="60" y="184" class="h">Underlay</text><text x="60" y="200" class="lbl">physical routed fabric</text>
<rect x="300" y="170" width="70" height="24" rx="5" class="node"/><text x="335" y="187" text-anchor="middle" class="s">Spine</text>
<rect x="215" y="224" width="70" height="24" rx="5" class="node"/><text x="250" y="241" text-anchor="middle" class="s">Leaf 1</text>
<rect x="450" y="224" width="70" height="24" rx="5" class="node"/><text x="485" y="241" text-anchor="middle" class="s">Leaf 40</text>
<line class="ul" x1="335" y1="194" x2="250" y2="224"/><line class="ul" x1="335" y1="194" x2="485" y2="224"/>
<line class="conn" x1="250" y1="94" x2="250" y2="224"/><line class="conn" x1="470" y1="94" x2="485" y2="224"/>
<text x="330" y="278" text-anchor="middle" class="lbl">The overlay rides on the underlay. A virtual wire up top is really packets routed across the fabric below.</text>
</g>
</svg>
<figcaption>Two planes. The overlay gives applications the Layer 2 segments they want, while the underlay stays a plain routed fabric.</figcaption>
</figure>

In the overlay, VM A and VM B appear to share one Ethernet segment, even though they sit under leaves 40 racks apart. The trick is **tunneling**. When VM A sends an Ethernet frame to VM B, the leaf under VM A wraps that whole frame inside a normal IP packet addressed to the leaf under VM B. The fabric routes that outer packet like any other, using all the ECMP paths from Part I. The far leaf unwraps it and delivers the original frame to VM B, which never knew its frame took a 40-rack detour inside an IP packet.

## Encapsulation, the core move

That wrap-and-route-and-unwrap is **encapsulation**. The original frame is the **inner** or tenant packet. The IP header the leaf adds is the **outer** header. The switches that add and remove this header are the **tunnel endpoints**.

<figure class="diagram">
<svg viewBox="0 0 680 170" style="width:100%;height:auto;max-width:680px" role="img" aria-label="A tenant frame enters a leaf, gets wrapped in an outer IP header, is routed across the fabric, and is unwrapped at the far leaf." xmlns="http://www.w3.org/2000/svg">
<style>
.o3{ font-family:var(--sans),sans-serif; }
.o3 .node{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.o3 .inner{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.3; }
.o3 .outer{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.o3 .fl{ stroke:var(--accent); stroke-width:2; fill:none; }
.o3 .s{ fill:var(--text-dim); font-size:13px; }
.o3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.o3 .mono{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
</style>
<g class="o3">
<rect x="20" y="62" width="70" height="30" rx="6" class="node"/><text x="55" y="81" text-anchor="middle" class="s">Leaf A</text>
<rect x="590" y="62" width="70" height="30" rx="6" class="node"/><text x="625" y="81" text-anchor="middle" class="s">Leaf B</text>
<rect x="108" y="66" width="70" height="22" rx="4" class="inner"/><text x="143" y="81" text-anchor="middle" class="mono">frame</text>
<text x="143" y="106" text-anchor="middle" class="lbl">tenant frame</text>
<rect x="240" y="58" width="190" height="38" rx="5" class="outer"/>
<rect x="340" y="64" width="82" height="26" rx="4" class="inner"/><text x="381" y="81" text-anchor="middle" class="mono">frame</text>
<text x="290" y="81" text-anchor="middle" class="mono">outer IP</text>
<text x="335" y="118" text-anchor="middle" class="lbl">wrapped, routed across the fabric by outer IP</text>
<rect x="500" y="66" width="70" height="22" rx="4" class="inner"/><text x="535" y="81" text-anchor="middle" class="mono">frame</text>
<text x="535" y="106" text-anchor="middle" class="lbl">unwrapped</text>
<line class="fl" x1="178" y1="77" x2="240" y2="77" marker-end="url(#ao3)"/><line class="fl" x1="430" y1="77" x2="500" y2="77" marker-end="url(#ao3)"/>
<defs><marker id="ao3" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Encapsulation in one line. The tenant frame travels inside an outer IP packet and emerges unchanged at the far leaf.</figcaption>
</figure>

This separation is the key insight of the whole book. The underlay only ever deals with the outer IP header, so it stays simple and stable. The overlay carries all the tenant complexity, VLANs, MACs, tenants, mobility, and it changes constantly without the underlay noticing. Each layer does one job.

## Choosing an encapsulation

Several encapsulation formats exist. NVGRE and STT were early contenders. Geneve is a newer, flexible format gaining ground, especially in software. But the format that became the industry default for hardware data center fabrics is **VXLAN**, short for Virtual Extensible LAN.

VXLAN won for concrete reasons. It carries a full Ethernet frame, so it delivers true Layer 2 adjacency. It uses UDP as its outer transport, which lets the fabric's ECMP hashing spread tunneled flows across all the spines. And it provides a 24-bit segment identifier, which shatters the old 4094-VLAN ceiling and allows about 16 million isolated segments, enough for the largest multi-tenant clouds.

The next chapter opens up the VXLAN packet byte by byte and walks a frame all the way across the fabric.
