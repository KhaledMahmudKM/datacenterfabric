# Chapter 11: Multihoming and Redundancy

A server connected to a single leaf loses its network if that leaf fails or is rebooted for maintenance. To avoid that, important servers connect to **two** leaves at once. This is **multihoming**, and it is where route types 1 and 4 finally earn their place. Multihoming is one of the more intricate parts of EVPN, so this chapter builds it up slowly with pictures.

## Dual-homing and the Ethernet Segment

A multihomed server has links to two leaves, bundled into one logical connection (a link aggregation group, or LAG) so the server treats them as a single interface. The pair of leaf links that attach to the same server is called an **Ethernet Segment**, identified by an **Ethernet Segment Identifier (ESI)**, a 10-byte value that both leaves configure identically for that server.

<figure class="diagram">
<svg viewBox="0 0 660 230" style="width:100%;height:auto;max-width:660px" role="img" aria-label="A server connects by a link aggregation group to Leaf 1 and Leaf 2, which share a common Ethernet Segment Identifier for that server." xmlns="http://www.w3.org/2000/svg">
<style>
.m1{ font-family:var(--sans),sans-serif; }
.m1 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.m1 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.4; }
.m1 .ln{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.m1 .es{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:5 4; }
.m1 .s{ fill:var(--text-dim); font-size:13px; }
.m1 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.m1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="m1">
<rect x="120" y="40" width="110" height="34" rx="6" class="leaf"/><text x="175" y="62" text-anchor="middle" class="s">Leaf 1</text>
<rect x="430" y="40" width="110" height="34" rx="6" class="leaf"/><text x="485" y="62" text-anchor="middle" class="s">Leaf 2</text>
<rect x="270" y="150" width="120" height="40" rx="6" class="node"/><text x="330" y="175" text-anchor="middle" class="s">server (LAG)</text>
<line class="ln" x1="200" y1="74" x2="300" y2="150"/><line class="ln" x1="460" y1="74" x2="360" y2="150"/>
<rect x="150" y="96" width="360" height="34" rx="8" class="es"/><text x="330" y="52" text-anchor="middle" class="s">Ethernet Segment</text><text x="330" y="70" text-anchor="middle" class="mono">same ESI on both leaves</text>
<text x="330" y="214" text-anchor="middle" class="lbl">One server, two leaves, one logical link. The ESI names this shared segment in EVPN.</text>
</g>
</svg>
<figcaption>A dual-homed server. The two leaf-facing links form one Ethernet Segment, given a shared ESI that both leaves advertise.</figcaption>
</figure>

EVPN multihoming is **standards-based** and does not need a special cabled peer link between the two leaves, which is the older MLAG approach. The two leaves coordinate entirely through BGP EVPN routes. That is a real operational win: any two leaves can jointly host a multihomed server without a dedicated inter-switch bond.

## Three problems multihoming must solve

Dual-homing introduces three hazards that the control plane has to handle. Keep these in mind as the route types appear.

<figure class="diagram">
<svg viewBox="0 0 680 190" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Three multihoming problems: avoiding duplicate BUM delivery, preventing loop-back of a server's own flooded frame, and fast failover when one leaf fails." xmlns="http://www.w3.org/2000/svg">
<style>
.m2{ font-family:var(--sans),sans-serif; }
.m2 .card{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.m2 .ic{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.m2 .t{ fill:var(--text); font-size:13px; font-weight:700; }
.m2 .s{ fill:var(--text-dim); font-size:13px; }
</style>
<g class="m2">
<rect x="20" y="30" width="210" height="120" rx="8" class="card"/><circle cx="55" cy="62" r="13" class="ic"/><text x="55" y="67" text-anchor="middle" class="t" fill="var(--accent)">1</text><text x="80" y="60" class="t">Duplicates</text><text x="30" y="92" class="s">BUM for the server could</text><text x="30" y="108" class="s">arrive on both leaves and</text><text x="30" y="124" class="s">be delivered to it twice.</text>
<rect x="240" y="30" width="210" height="120" rx="8" class="card"/><circle cx="275" cy="62" r="13" class="ic"/><text x="275" y="67" text-anchor="middle" class="t" fill="var(--accent)">2</text><text x="300" y="60" class="t">Loop-back</text><text x="250" y="92" class="s">A frame the server flooded</text><text x="250" y="108" class="s">could come back to it via</text><text x="250" y="124" class="s">the other leaf.</text>
<rect x="460" y="30" width="200" height="120" rx="8" class="card"/><circle cx="495" cy="62" r="13" class="ic"/><text x="495" y="67" text-anchor="middle" class="t" fill="var(--accent)">3</text><text x="520" y="60" class="t">Failover</text><text x="470" y="92" class="s">If one leaf dies, every</text><text x="470" y="108" class="s">remote VTEP must switch</text><text x="470" y="124" class="s">to the survivor fast.</text>
</g>
</svg>
<figcaption>The three multihoming hazards. Types 1 and 4 exist to solve exactly these.</figcaption>
</figure>

## Type 4 and the designated forwarder

The first two problems are about BUM traffic. If both leaves forwarded every broadcast to the server, it would see duplicates. So the two leaves must agree that only **one** of them delivers BUM to the segment at a time. That chosen leaf is the **Designated Forwarder (DF)**.

The leaves discover each other and elect the DF using the **type 4 route**, the **Ethernet Segment** route. Every leaf attached to an ESI advertises a type 4 route for it. The leaves that see a shared ESI know they are peers on that segment and run a deterministic election to pick the DF.

<figure class="diagram">
<svg viewBox="0 0 680 230" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Leaf 1 and Leaf 2 both advertise a type 4 route for the shared ESI, discover they are peers, and elect Leaf 1 as designated forwarder for BUM traffic." xmlns="http://www.w3.org/2000/svg">
<style>
.m3{ font-family:var(--sans),sans-serif; }
.m3 .df{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.8; }
.m3 .ndf{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.m3 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.m3 .adv{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:5 4; }
.m3 .bum{ stroke:var(--accent); stroke-width:2.4; fill:none; }
.m3 .blk{ stroke:var(--text-faint); stroke-width:2; fill:none; stroke-dasharray:3 4; }
.m3 .s{ fill:var(--text-dim); font-size:13px; }
.m3 .tag{ fill:var(--accent); font-size:13px; font-weight:700; }
.m3 .no{ fill:var(--text-faint); font-size:13px; font-weight:700; }
.m3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="m3">
<rect x="110" y="40" width="120" height="40" rx="6" class="df"/><text x="170" y="60" text-anchor="middle" class="s">Leaf 1</text><text x="170" y="74" text-anchor="middle" class="tag">DF</text>
<rect x="430" y="40" width="120" height="40" rx="6" class="ndf"/><text x="490" y="60" text-anchor="middle" class="s">Leaf 2</text><text x="490" y="74" text-anchor="middle" class="no">non-DF</text>
<line class="adv" x1="230" y1="60" x2="430" y2="60" marker-end="url(#am3)" marker-start="url(#am3b)"/>
<text x="330" y="36" text-anchor="middle" class="lbl">type 4 routes for the shared ESI, then DF election</text>
<rect x="270" y="150" width="120" height="40" rx="6" class="node"/><text x="330" y="175" text-anchor="middle" class="s">server</text>
<line class="bum" x1="170" y1="80" x2="310" y2="150" marker-end="url(#am3)"/><text x="195" y="128" class="tag">BUM</text>
<line class="blk" x1="490" y1="80" x2="350" y2="150"/><text x="450" y="128" class="no">BUM blocked</text>
<text x="330" y="214" text-anchor="middle" class="lbl">Only the DF sends BUM to the server, so it never sees a duplicate broadcast.</text>
<defs><marker id="am3" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker><marker id="am3b" markerWidth="8" markerHeight="8" refX="2" refY="4" orient="auto"><path d="M8 0 L0 4 L8 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>DF election through type 4 routes. Only the designated forwarder delivers BUM to the multihomed server, which eliminates duplicate broadcasts.</figcaption>
</figure>

## Split-horizon stops the loop-back

The second hazard is subtler. Suppose the server floods a frame and it enters the fabric through Leaf 1. That frame will be replicated to all VTEPs in the VNI, including Leaf 2, the server's other leaf. If Leaf 2 then forwarded it down to the server, the server would receive its own frame back.

EVPN prevents this with **split-horizon filtering**. A frame that a leaf received from the fabric and that is tagged as coming from a given Ethernet Segment is never sent back out that same segment's local links. The type 1 route (described next) carries the ESI label that lets a leaf recognize such frames and drop them at the door.

<figure class="diagram">
<svg viewBox="0 0 660 220" style="width:100%;height:auto;max-width:660px" role="img" aria-label="A frame flooded by the server via Leaf 1 reaches Leaf 2 through the fabric, but split-horizon filtering stops Leaf 2 from sending it back to the server." xmlns="http://www.w3.org/2000/svg">
<style>
.m4{ font-family:var(--sans),sans-serif; }
.m4 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.m4 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.m4 .fl{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.m4 .blk{ stroke:var(--text-faint); stroke-width:2.2; fill:none; stroke-dasharray:3 4; }
.m4 .s{ fill:var(--text-dim); font-size:13px; }
.m4 .no{ fill:var(--text-faint); font-size:13px; font-weight:700; }
.m4 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="m4">
<rect x="90" y="40" width="110" height="36" rx="6" class="leaf"/><text x="145" y="62" text-anchor="middle" class="s">Leaf 1</text>
<rect x="450" y="40" width="110" height="36" rx="6" class="leaf"/><text x="505" y="62" text-anchor="middle" class="s">Leaf 2</text>
<rect x="270" y="150" width="120" height="40" rx="6" class="node"/><text x="330" y="175" text-anchor="middle" class="s">server</text>
<line class="fl" x1="300" y1="150" x2="170" y2="76" marker-end="url(#am4)"/><text x="222" y="128" text-anchor="end" class="lbl">floods up</text>
<line class="fl" x1="200" y1="58" x2="450" y2="58" marker-end="url(#am4)"/><text x="325" y="46" text-anchor="middle" class="lbl">replicated across fabric</text>
<line class="blk" x1="490" y1="76" x2="360" y2="150"/><text x="470" y="120" class="no">split-horizon</text><text x="470" y="136" class="no">drops it</text>
<text x="330" y="208" text-anchor="middle" class="lbl">The server never receives the frame it just sent. The loop is cut at Leaf 2.</text>
<defs><marker id="am4" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Split-horizon filtering. A frame returning to the segment it originated from is dropped, so a multihomed server never loops its own flooded traffic.</figcaption>
</figure>

## Type 1: aliasing and fast failover

The type 1 route, **Ethernet Auto-Discovery (A-D)**, does two jobs, both about using both leaves well.

**Aliasing.** A remote VTEP may have learned a given server MAC (through a type 2 route) from only one of the two leaves, because that is the leaf that happened to see its traffic. But the server is reachable through both. The **per-ES type 1 route** tells remote VTEPs that both leaves can reach everything on that Ethernet Segment, so they can **load-balance** unicast across both uplinks instead of pinning all traffic to one leaf.

<figure class="diagram">
<svg viewBox="0 0 680 210" style="width:100%;height:auto;max-width:680px" role="img" aria-label="A remote VTEP load-balances traffic for a dual-homed server across both Leaf 1 and Leaf 2 thanks to the type 1 aliasing route." xmlns="http://www.w3.org/2000/svg">
<style>
.m5{ font-family:var(--sans),sans-serif; }
.m5 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.m5 .vtep{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.m5 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.m5 .fl{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.m5 .s{ fill:var(--text-dim); font-size:13px; }
.m5 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="m5">
<rect x="40" y="86" width="100" height="40" rx="6" class="vtep"/><text x="90" y="110" text-anchor="middle" class="s">remote VTEP</text>
<rect x="320" y="40" width="100" height="36" rx="6" class="leaf"/><text x="370" y="62" text-anchor="middle" class="s">Leaf 1</text>
<rect x="320" y="134" width="100" height="36" rx="6" class="leaf"/><text x="370" y="156" text-anchor="middle" class="s">Leaf 2</text>
<rect x="520" y="86" width="140" height="40" rx="6" class="node"/><text x="590" y="110" text-anchor="middle" class="s">dual-homed server</text>
<line class="fl" x1="140" y1="100" x2="320" y2="60" marker-end="url(#am5)"/><line class="fl" x1="140" y1="112" x2="320" y2="152" marker-end="url(#am5)"/>
<line class="fl" x1="420" y1="58" x2="520" y2="98" marker-end="url(#am5)"/><line class="fl" x1="420" y1="152" x2="520" y2="114" marker-end="url(#am5)"/>
<text x="340" y="104" text-anchor="middle" class="lbl">both paths used</text>
<text x="340" y="196" text-anchor="middle" class="lbl">Type 1 aliasing lets the remote VTEP use both leaves, not just the one that advertised the MAC.</text>
<defs><marker id="am5" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Aliasing. The type 1 route advertises that the whole segment is reachable via both leaves, so remote traffic load-balances across them.</figcaption>
</figure>

**Fast failover, through mass withdrawal.** If Leaf 1 fails, every remote VTEP must stop sending that server's traffic to Leaf 1. Withdrawing a type 2 route for every affected MAC one by one would be slow when thousands of MACs sit on the segment. Instead, Leaf 1's failure withdraws a **single per-ES type 1 route**, and remote VTEPs instantly treat **all** MACs on that segment as reachable only through Leaf 2. One withdrawal, not thousands. This is often called **mass withdrawal**, and it is why EVPN failover is quick even at scale.

<figure class="diagram">
<svg viewBox="0 0 680 216" style="width:100%;height:auto;max-width:680px" role="img" aria-label="When Leaf 1 fails, a single type 1 route withdrawal tells remote VTEPs to move all of the segment's MACs to Leaf 2 at once." xmlns="http://www.w3.org/2000/svg">
<style>
.m6{ font-family:var(--sans),sans-serif; }
.m6 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.m6 .dead{ fill:var(--panel); stroke:var(--text-faint); stroke-width:1.4; stroke-dasharray:4 4; }
.m6 .vtep{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.m6 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.m6 .fl{ stroke:var(--accent); stroke-width:2.4; fill:none; }
.m6 .x{ stroke:var(--text-faint); stroke-width:2.2; fill:none; }
.m6 .s{ fill:var(--text-dim); font-size:13px; }
.m6 .no{ fill:var(--text-faint); font-size:13px; font-weight:700; }
.m6 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="m6">
<rect x="40" y="86" width="100" height="40" rx="6" class="vtep"/><text x="90" y="110" text-anchor="middle" class="s">remote VTEP</text>
<rect x="320" y="36" width="100" height="36" rx="6" class="dead"/><text x="370" y="58" text-anchor="middle" class="no">Leaf 1 down</text>
<rect x="320" y="138" width="100" height="36" rx="6" class="leaf"/><text x="370" y="160" text-anchor="middle" class="s">Leaf 2</text>
<rect x="520" y="86" width="110" height="40" rx="6" class="node"/><text x="575" y="110" text-anchor="middle" class="s">server</text>
<line class="x" x1="150" y1="98" x2="312" y2="56"/><line class="x" x1="230" y1="60" x2="245" y2="90" /><line class="x" x1="245" y1="60" x2="230" y2="90"/>
<line class="fl" x1="140" y1="112" x2="320" y2="156" marker-end="url(#am6)"/><line class="fl" x1="420" y1="156" x2="520" y2="114" marker-end="url(#am6)"/>
<text x="40" y="162" class="lbl">one type 1 withdrawal moves</text><text x="40" y="179" class="lbl">ALL MACs to Leaf 2</text>
<text x="340" y="206" text-anchor="middle" class="lbl">Failover is a single route change, not thousands. That is what makes it fast.</text>
<defs><marker id="am6" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Mass withdrawal. One type 1 route withdrawal fails the whole segment over to the surviving leaf, so convergence does not depend on the number of hosts.</figcaption>
</figure>

## Pulling the route types together

Now all five route types have a job, and it is worth seeing them in one summary before we move to design:

- **Type 2** advertises each host (MAC, and optionally IP).
- **Type 3** discovers which VTEPs are in a VNI, for BUM replication.
- **Type 5** advertises subnets and external prefixes, for routing.
- **Type 4** discovers segment peers and elects the designated forwarder.
- **Type 1** provides aliasing and fast mass-withdrawal failover for multihomed segments.

## A note on complexity

Multihoming is the most complex corner of EVPN, and in practice many fabrics keep it simple. Some use EVPN multihoming as described here. Others dual-home servers using a vendor's MLAG feature and run EVPN only between the resulting logical switches. And a great many servers are simply single-homed, relying on the application layer for redundancy instead. The right amount of multihoming is a design decision, which is exactly the kind of decision the next chapter is about.
