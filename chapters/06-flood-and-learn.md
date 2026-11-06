# Chapter 6: Flood-and-Learn VXLAN

Chapter 5 left one question open: how does a VTEP know which remote VTEP holds a given MAC address? The original VXLAN design, in RFC 7348, answered it the same way a plain Ethernet switch does, with **flooding and learning**. This chapter shows how that works, and why it does not scale. Understanding its limits is the clearest way to see what EVPN was created to fix.

## How a plain switch learns

Recall how a single Ethernet switch builds its MAC table. It starts knowing nothing. When a frame arrives, the switch records the source MAC and the port it came in on, which is **learning**. When it must forward a frame to a destination it has not learned yet, it **floods** the frame out every port in that VLAN and lets the reply teach it where the destination is. Broadcasts and multicasts are flooded by definition.

VXLAN flood-and-learn applies this same logic, except the "ports" are now remote VTEPs reached across the fabric.

## BUM traffic

The traffic that must be flooded has a name: **BUM**, for **Broadcast, Unknown-unicast, and Multicast**.

- **Broadcast**, such as an ARP request asking "who has this IP?"
- **Unknown unicast**, a frame to a MAC the ingress VTEP has not learned yet.
- **Multicast**, frames to a multicast group.

In a VXLAN segment, when a VTEP receives BUM traffic from a local server, it must deliver a copy to **every other VTEP that has a member in the same VNI**. There are two ways to do that.

<figure class="diagram">
<svg viewBox="0 0 700 278" style="width:100%;height:auto;max-width:700px" role="img" aria-label="Two methods for delivering BUM traffic: head-end replication where the source VTEP sends a separate copy to each remote VTEP, and multicast where the fabric replicates one copy." xmlns="http://www.w3.org/2000/svg">
<style>
.b1{ font-family:var(--sans),sans-serif; }
.b1 .vtep{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.b1 .src{ fill:var(--accent); stroke:var(--accent); stroke-width:1.5; }
.b1 .fab{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.b1 .cp{ stroke:var(--accent); stroke-width:2; fill:none; }
.b1 .one{ stroke:var(--text-faint); stroke-width:2; fill:none; }
.b1 .h{ fill:var(--text); font-size:15px; font-weight:700; }
.b1 .s{ fill:var(--text-dim); font-size:13px; }
.b1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="b1">
<text x="175" y="22" text-anchor="middle" class="h">Head-end replication</text><text x="175" y="38" text-anchor="middle" class="lbl">source makes N copies</text>
<rect x="30" y="110" width="80" height="34" rx="6" class="src"/><text x="70" y="131" text-anchor="middle" class="s" style="fill:var(--bg)">VTEP 1</text>
<rect x="250" y="56" width="70" height="28" rx="6" class="vtep"/><text x="285" y="74" text-anchor="middle" class="s">VTEP 2</text>
<rect x="250" y="112" width="70" height="28" rx="6" class="vtep"/><text x="285" y="130" text-anchor="middle" class="s">VTEP 3</text>
<rect x="250" y="168" width="70" height="28" rx="6" class="vtep"/><text x="285" y="186" text-anchor="middle" class="s">VTEP 4</text>
<line class="cp" x1="110" y1="120" x2="250" y2="70"/><line class="cp" x1="110" y1="127" x2="250" y2="126"/><line class="cp" x1="110" y1="134" x2="250" y2="182"/>
<text x="175" y="216" text-anchor="middle" class="lbl">3 copies leave VTEP 1.</text><text x="175" y="233" text-anchor="middle" class="lbl">With N VTEPs, N-1 copies.</text>
<text x="535" y="22" text-anchor="middle" class="h">Multicast</text><text x="535" y="38" text-anchor="middle" class="lbl">fabric replicates one copy</text>
<rect x="400" y="110" width="80" height="34" rx="6" class="src"/><text x="440" y="131" text-anchor="middle" class="s" style="fill:var(--bg)">VTEP 1</text>
<rect x="540" y="104" width="60" height="28" rx="6" class="fab"/><text x="570" y="122" text-anchor="middle" class="s">spine</text>
<rect x="640" y="56" width="50" height="28" rx="6" class="vtep"/><rect x="640" y="112" width="50" height="28" rx="6" class="vtep"/><rect x="640" y="168" width="50" height="28" rx="6" class="vtep"/>
<line class="one" x1="480" y1="125" x2="540" y2="120"/><line class="cp" x1="600" y1="112" x2="640" y2="72"/><line class="cp" x1="600" y1="118" x2="640" y2="126"/><line class="cp" x1="600" y1="124" x2="640" y2="182"/>
<text x="545" y="216" text-anchor="middle" class="lbl">1 copy leaves VTEP 1; the fabric copies it.</text><text x="545" y="233" text-anchor="middle" class="lbl">Needs multicast routing (PIM).</text>
<text x="350" y="264" text-anchor="middle" class="s">Both deliver BUM to every VTEP in the VNI. Neither is free.</text>
</g>
</svg>
<figcaption>Two ways to flood BUM traffic across the fabric. Head-end replication burdens the source VTEP; multicast burdens the fabric with group state.</figcaption>
</figure>

**Head-end replication**, also called ingress replication, has the source VTEP make one unicast copy of the frame for each remote VTEP in the VNI and send them all itself. It needs no special fabric support, but the source does all the work, and the copies multiply with the number of VTEPs.

**Multicast** has the source send a single copy to a multicast group, and the fabric's multicast routing replicates it to the members. This is efficient on the source, but it requires running multicast (PIM) across the underlay, which many operators would rather avoid for its complexity.

## Data-plane learning over the tunnel

Once BUM delivery exists, learning falls out of it, exactly as in a plain switch. When a VTEP decapsulates a frame, it reads the **inner source MAC** and the **outer source IP** (the sending VTEP's loopback) and records the pair: "this MAC lives behind that VTEP." Future unicast frames to that MAC can then be sent straight to the right VTEP instead of flooded.

<figure class="diagram">
<svg viewBox="0 0 700 262" style="width:100%;height:auto;max-width:700px" role="img" aria-label="VM A sends a broadcast ARP, which floods to all VTEPs. VM B replies, and along the way VTEPs learn each other's MAC-to-VTEP mapping from the data plane." xmlns="http://www.w3.org/2000/svg">
<style>
.b2{ font-family:var(--sans),sans-serif; }
.b2 .vtep{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.b2 .node{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.b2 .flood{ stroke:var(--text-faint); stroke-width:1.8; fill:none; stroke-dasharray:5 4; }
.b2 .reply{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.b2 .s{ fill:var(--text-dim); font-size:13px; }
.b2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.b2 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.b2 .step{ fill:var(--text); font-size:13px; font-weight:700; }
</style>
<g class="b2">
<rect x="20" y="96" width="70" height="30" rx="6" class="node"/><text x="55" y="116" text-anchor="middle" class="s">VM A</text>
<rect x="120" y="86" width="100" height="50" rx="6" class="vtep"/><text x="170" y="116" text-anchor="middle" class="s">VTEP 1</text>
<rect x="420" y="30" width="100" height="40" rx="6" class="vtep"/><text x="470" y="55" text-anchor="middle" class="s">VTEP 2</text>
<rect x="420" y="152" width="100" height="40" rx="6" class="vtep"/><text x="470" y="177" text-anchor="middle" class="s">VTEP 3</text>
<rect x="580" y="157" width="70" height="30" rx="6" class="node"/><text x="615" y="177" text-anchor="middle" class="s">VM B</text>
<line class="flood" x1="220" y1="100" x2="420" y2="50"/><line class="flood" x1="220" y1="118" x2="420" y2="162"/>
<text x="226" y="36" class="step">1 broadcast ARP, flooded</text>
<line class="reply" x1="580" y1="172" x2="520" y2="172"/><line class="reply" x1="420" y1="182" x2="220" y2="130" marker-end="url(#ab2)"/>
<text x="226" y="214" class="step">2 unicast reply from VM B</text>
<text x="170" y="168" text-anchor="middle" class="mono">learns: VM B at VTEP 3</text>
<text x="470" y="212" text-anchor="middle" class="mono">VTEP 3 learned:</text><text x="470" y="228" text-anchor="middle" class="mono">VM A at VTEP 1</text>
<text x="350" y="252" text-anchor="middle" class="lbl">Learning is a side effect of traffic. Until a MAC is learned, frames to it are flooded.</text>
<defs><marker id="ab2" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Data-plane learning. A VTEP learns a remote MAC only when it receives a frame from it, so the first packet to any unknown host is always flooded.</figcaption>
</figure>

This works, and for small VXLAN deployments it is perfectly serviceable. It also needs no new protocol: the VTEPs just encapsulate, flood, and learn.

## Why flood-and-learn does not scale

The trouble shows up as the fabric grows. The problems all trace back to one root cause: there is **no control plane**. The VTEPs never talk to each other about who knows what. All knowledge comes from watching traffic, so the design inherits every weakness of that approach.

<figure class="diagram">
<svg viewBox="0 0 680 240" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Four drawbacks of flood-and-learn: constant flooding wastes bandwidth, learning is reactive and slow, failures cause black holes until timeout, and multicast adds fabric complexity." xmlns="http://www.w3.org/2000/svg">
<style>
.b3{ font-family:var(--sans),sans-serif; }
.b3 .card{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.b3 .ic{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.b3 .t{ fill:var(--text); font-size:13px; font-weight:700; }
.b3 .s{ fill:var(--text-dim); font-size:13px; }
</style>
<g class="b3">
<rect x="20" y="24" width="315" height="92" rx="8" class="card"/><circle cx="52" cy="54" r="14" class="ic"/><text x="80" y="50" class="t">Flooding wastes bandwidth</text><text x="80" y="70" class="s">Every unknown and every broadcast hits</text><text x="80" y="86" class="s">every VTEP in the segment, all the time.</text>
<rect x="345" y="24" width="315" height="92" rx="8" class="card"/><circle cx="377" cy="54" r="14" class="ic"/><text x="405" y="50" class="t">Learning is reactive</text><text x="405" y="70" class="s">A VTEP knows a host only after seeing</text><text x="405" y="86" class="s">its traffic. The first packet always floods.</text>
<rect x="20" y="128" width="315" height="92" rx="8" class="card"/><circle cx="52" cy="158" r="14" class="ic"/><text x="80" y="154" class="t">Failures cause black holes</text><text x="80" y="174" class="s">If a host moves or a VTEP fails, stale</text><text x="80" y="190" class="s">entries drop traffic until they age out.</text>
<rect x="345" y="128" width="315" height="92" rx="8" class="card"/><circle cx="377" cy="158" r="14" class="ic"/><text x="405" y="154" class="t">Multicast is complex</text><text x="405" y="174" class="s">Efficient BUM needs PIM in the underlay,</text><text x="405" y="190" class="s">which operators work hard to avoid.</text>
</g>
</svg>
<figcaption>The four costs of having no control plane. Each one gets worse as the number of VTEPs and hosts grows.</figcaption>
</figure>

To make the first weakness concrete: **ARP** is relentless. Every host in a subnet periodically broadcasts ARP requests to resolve its neighbors. In a large VNI spanning many VTEPs, that is a steady rain of broadcast frames, each one flooded to every VTEP in the segment, consuming bandwidth and CPU everywhere even when almost no VTEP cares about the answer.

## The missing piece

Step back and the gap is obvious. A plain switch floods and learns because it has no other way to discover where hosts are. But our fabric already runs a sophisticated control plane for the underlay: **BGP**, from Chapter 3. BGP exists precisely to distribute reachability information efficiently, proactively, and at scale.

The natural question is: what if the VTEPs used BGP to **tell each other** which MAC addresses and which hosts they have, instead of making everyone learn it the hard way by flooding? Then a VTEP would know where every host is before the first packet, ARP could be answered locally instead of flooded, and host moves could be announced instantly rather than discovered after a timeout.

That is exactly what **EVPN** does. It is a BGP-based control plane for the VXLAN data plane we built in Chapter 5. Part III is devoted to it, starting with its fundamentals in the next chapter.
