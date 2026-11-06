# Chapter 7: EVPN Fundamentals

**EVPN**, Ethernet VPN, is the control plane that fixes everything Chapter 6 complained about. It uses BGP to distribute host and segment information between VTEPs, so they no longer have to flood and learn. EVPN was first defined in **RFC 7432**, and **RFC 8365** describes how to run it over a VXLAN data plane, which is the combination, often written **EVPN-VXLAN**, that this book is about.

This chapter builds the mental model. The specific route types come in Chapter 8.

## Control plane and data plane, cleanly split

The single most important idea in Part III is the separation of the two planes.

<figure class="diagram">
<svg viewBox="0 0 680 272" style="width:100%;height:auto;max-width:680px" role="img" aria-label="EVPN is the control plane that decides where hosts are, while VXLAN is the data plane that forwards packets. BGP carries the control plane; the fabric carries the data plane." xmlns="http://www.w3.org/2000/svg">
<style>
.e1{ font-family:var(--sans),sans-serif; }
.e1 .cp{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.e1 .dp{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.e1 .node{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.e1 .ctl{ stroke:var(--accent); stroke-width:2; fill:none; stroke-dasharray:5 4; }
.e1 .dat{ stroke:var(--text-faint); stroke-width:2; fill:none; }
.e1 .h{ fill:var(--text); font-size:15px; font-weight:700; }
.e1 .s{ fill:var(--text-dim); font-size:13px; }
.e1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="e1">
<rect x="30" y="20" width="620" height="112" rx="8" class="cp" opacity="0.22"/>
<text x="48" y="42" class="h">Control plane: EVPN over BGP</text><text x="48" y="60" class="lbl">decides where every host and segment lives, in advance</text>
<rect x="60" y="88" width="110" height="30" rx="6" class="node"/><text x="115" y="108" text-anchor="middle" class="s">VTEP 1</text>
<rect x="510" y="88" width="110" height="30" rx="6" class="node"/><text x="565" y="108" text-anchor="middle" class="s">VTEP 2</text>
<line class="ctl" x1="170" y1="103" x2="510" y2="103" marker-end="url(#ae1)" marker-start="url(#ae1b)"/>
<text x="340" y="95" text-anchor="middle" class="lbl">"I have MAC x in VNI 10"</text>
<rect x="30" y="148" width="620" height="112" rx="8" class="dp"/>
<text x="48" y="170" class="h">Data plane: VXLAN over the fabric</text><text x="48" y="188" class="lbl">forwards the actual frames, using what the control plane already told it</text>
<rect x="60" y="216" width="110" height="30" rx="6" class="node"/><text x="115" y="236" text-anchor="middle" class="s">VTEP 1</text>
<rect x="510" y="216" width="110" height="30" rx="6" class="node"/><text x="565" y="236" text-anchor="middle" class="s">VTEP 2</text>
<line class="dat" x1="170" y1="231" x2="510" y2="231" marker-end="url(#ae1c)"/>
<text x="340" y="223" text-anchor="middle" class="lbl">encapsulated frames, straight to the right VTEP</text>
<defs>
<marker id="ae1" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9 z" fill="var(--accent)"/></marker>
<marker id="ae1b" markerWidth="9" markerHeight="9" refX="2" refY="4.5" orient="auto"><path d="M9 0 L0 4.5 L9 9 z" fill="var(--accent)"/></marker>
<marker id="ae1c" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9 z" fill="var(--text-faint)"/></marker>
</defs>
</g>
</svg>
<figcaption>The clean split. EVPN (control) learns and advertises where things are; VXLAN (data) forwards frames. Chapter 5 built the lower box; Part III builds the upper one.</figcaption>
</figure>

Flood-and-learn mixed these two jobs, forcing the data plane to also do the discovery. EVPN separates them. The control plane learns host locations once and advertises them to every VTEP, proactively. The data plane then forwards with full knowledge and never has to flood to discover anything.

## Learn locally, advertise with BGP

A VTEP running EVPN still learns about its **local** hosts the ordinary way, from the data plane or from ARP, because those hosts are directly attached. The change is what happens next. Instead of waiting for remote VTEPs to discover that host by receiving its traffic, the VTEP immediately **advertises** it in a BGP update: "MAC address x, in segment VNI 10, is reachable through me, and here is my VTEP IP."

<figure class="diagram">
<svg viewBox="0 0 700 220" style="width:100%;height:auto;max-width:700px" role="img" aria-label="A VTEP learns a local host, then advertises it via BGP to all other VTEPs, which install it without ever seeing the host's traffic." xmlns="http://www.w3.org/2000/svg">
<style>
.e2{ font-family:var(--sans),sans-serif; }
.e2 .vtep{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.e2 .rr{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.e2 .node{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.e2 .adv{ stroke:var(--accent); stroke-width:2; fill:none; }
.e2 .loc{ stroke:var(--text-faint); stroke-width:1.6; fill:none; }
.e2 .s{ fill:var(--text-dim); font-size:13px; }
.e2 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.e2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.e2 .step{ fill:var(--text); font-size:13px; font-weight:700; }
</style>
<g class="e2">
<rect x="30" y="92" width="56" height="26" rx="5" class="node"/><text x="58" y="109" text-anchor="middle" class="s">host</text>
<rect x="110" y="84" width="84" height="42" rx="6" class="vtep"/><text x="152" y="109" text-anchor="middle" class="s">VTEP 1</text>
<line class="loc" x1="86" y1="105" x2="110" y2="105" marker-end="url(#ae2)"/>
<text x="150" y="72" text-anchor="middle" class="step">1 learn local</text>
<rect x="310" y="86" width="130" height="38" rx="6" class="rr"/><text x="375" y="110" text-anchor="middle" class="s">route reflector</text>
<line class="adv" x1="194" y1="105" x2="310" y2="105" marker-end="url(#ae2)"/>
<text x="252" y="94" text-anchor="middle" class="mono">BGP update</text>
<rect x="560" y="40" width="84" height="34" rx="6" class="vtep"/><text x="602" y="61" text-anchor="middle" class="s">VTEP 2</text>
<rect x="560" y="92" width="84" height="34" rx="6" class="vtep"/><text x="602" y="113" text-anchor="middle" class="s">VTEP 3</text>
<rect x="560" y="144" width="84" height="34" rx="6" class="vtep"/><text x="602" y="165" text-anchor="middle" class="s">VTEP 4</text>
<line class="adv" x1="440" y1="100" x2="560" y2="57" marker-end="url(#ae2)"/><line class="adv" x1="440" y1="105" x2="560" y2="109" marker-end="url(#ae2)"/><line class="adv" x1="440" y1="110" x2="560" y2="161" marker-end="url(#ae2)"/>
<text x="490" y="30" text-anchor="middle" class="step">2 reflect to all VTEPs</text>
<text x="350" y="205" text-anchor="middle" class="lbl">Every VTEP now knows the host before it sends a single frame. No flooding needed to discover it.</text>
<defs><marker id="ae2" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>EVPN turns host discovery into a BGP advertisement. Learn once, locally, then tell everyone through BGP.</figcaption>
</figure>

Notice the **route reflector** in the middle. With many VTEPs, having every VTEP peer directly with every other is a lot of sessions. Instead, VTEPs peer with one or two route reflectors, often the spines, which receive each advertisement once and reflect it to all the others. This keeps the number of BGP sessions linear in the number of VTEPs rather than quadratic.

## The new BGP address family

BGP can carry many kinds of reachability, each in its own **address family**, identified by an AFI (Address Family Identifier) and SAFI (Subsequent Address Family Identifier). IPv4 unicast routes, the ones the underlay uses, are one family. EVPN is another: **AFI 25 (L2VPN) with SAFI 70 (EVPN)**. The same BGP process and often the very same peering sessions carry both families at once, each tagged with its own AFI/SAFI.

<figure class="diagram">
<svg viewBox="0 0 660 190" style="width:100%;height:auto;max-width:660px" role="img" aria-label="One BGP session between two switches carries two address families at once: IPv4 unicast for the underlay and L2VPN EVPN for the overlay." xmlns="http://www.w3.org/2000/svg">
<style>
.e3{ font-family:var(--sans),sans-serif; }
.e3 .node{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.e3 .pipe{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.e3 .af1{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.e3 .af2{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.e3 .s{ fill:var(--text-dim); font-size:13px; }
.e3 .mono{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.e3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="e3">
<rect x="20" y="70" width="90" height="44" rx="6" class="node"/><text x="65" y="97" text-anchor="middle" class="s">Leaf</text>
<rect x="550" y="70" width="90" height="44" rx="6" class="node"/><text x="595" y="97" text-anchor="middle" class="s">Spine (RR)</text>
<rect x="126" y="56" width="408" height="72" rx="10" class="pipe"/>
<text x="330" y="48" text-anchor="middle" class="lbl">one BGP session</text>
<rect x="136" y="68" width="388" height="24" rx="5" class="af1"/><text x="330" y="84" text-anchor="middle" class="mono">AFI 1 / SAFI 1   IPv4 unicast   (underlay loopbacks)</text>
<rect x="136" y="98" width="388" height="24" rx="5" class="af2"/><text x="330" y="114" text-anchor="middle" class="mono">AFI 25 / SAFI 70   L2VPN EVPN   (overlay hosts)</text>
<text x="330" y="160" text-anchor="middle" class="lbl">Underlay and overlay share the same BGP plumbing. EVPN is just another family on it.</text>
</g>
</svg>
<figcaption>EVPN is carried as a BGP address family, AFI 25 / SAFI 70. It rides the same sessions that already build the underlay.</figcaption>
</figure>

This reuse is a big part of why BGP won the underlay in Chapter 3. One protocol, one set of sessions, two jobs.

## Three terms you need: EVI, RD, RT

EVPN borrows three constructs from the older BGP/MPLS VPN world. They let one fabric carry many isolated tenants without their addresses colliding.

- **EVI (EVPN Instance).** A tenant's virtual network, roughly "one VNI's worth of EVPN state." Each isolated segment is an EVI.
- **RD (Route Distinguisher).** A 64-bit value prepended to each advertised address to make it **globally unique**. Two tenants might both use MAC or IP ranges that overlap, or both advertise the same host address. Giving each its own RD means the routes stay distinct inside BGP even when the underlying addresses are identical. A common scheme sets the RD from the VTEP's loopback plus the VNI, so it is unique per VTEP per segment.
- **RT (Route Target).** A BGP extended community that controls **import and export**. When a VTEP advertises a route, it tags it with an export RT. When a VTEP receives routes, it imports only those carrying an RT it is configured to accept. RTs are how a VTEP decides which tenant a route belongs to and whether it cares.

<figure class="diagram">
<svg viewBox="0 0 700 230" style="width:100%;height:auto;max-width:700px" role="img" aria-label="An advertised EVPN route carries a route distinguisher to stay unique and a route target that controls which VTEPs import it, keeping two tenants isolated." xmlns="http://www.w3.org/2000/svg">
<style>
.e4{ font-family:var(--sans),sans-serif; }
.e4 .route{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.e4 .rd{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.e4 .rt{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.e4 .vtepA{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.e4 .vtepB{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.e4 .ln{ stroke:var(--accent); stroke-width:1.8; fill:none; }
.e4 .blk{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.e4 .s{ fill:var(--text-dim); font-size:13px; }
.e4 .mono{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.e4 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.e4 .ok{ fill:var(--accent); font-size:13px; font-weight:700; }
.e4 .no{ fill:var(--text-faint); font-size:13px; font-weight:700; }
</style>
<g class="e4">
<rect x="180" y="24" width="340" height="52" rx="8" class="route"/>
<rect x="192" y="36" width="150" height="28" rx="5" class="rd"/><text x="267" y="54" text-anchor="middle" class="mono">RD 10.0.1.1:10</text>
<rect x="352" y="36" width="156" height="28" rx="5" class="rt"/><text x="430" y="54" text-anchor="middle" class="mono">RT 65000:10</text>
<text x="350" y="92" text-anchor="middle" class="lbl">one EVPN route: RD keeps it unique, RT says which tenant it belongs to</text>
<rect x="20" y="136" width="160" height="48" rx="6" class="vtepA"/><text x="100" y="156" text-anchor="middle" class="s">VTEP in tenant</text><text x="100" y="174" text-anchor="middle" class="mono">imports RT 65000:10</text>
<rect x="280" y="140" width="130" height="40" rx="6" class="vtepA"/><text x="345" y="165" text-anchor="middle" class="s">advertiser</text>
<rect x="520" y="136" width="160" height="48" rx="6" class="vtepB"/><text x="600" y="156" text-anchor="middle" class="s">other tenant</text><text x="600" y="174" text-anchor="middle" class="mono">imports RT 65000:20</text>
<line class="ln" x1="280" y1="160" x2="180" y2="160" marker-end="url(#ae4)"/><text x="230" y="151" text-anchor="middle" class="ok">imported</text>
<line class="blk" x1="410" y1="160" x2="520" y2="160"/><text x="465" y="151" text-anchor="middle" class="no">ignored</text>
<text x="350" y="214" text-anchor="middle" class="lbl">Same route, two outcomes. RT matching is how EVPN keeps tenants isolated on a shared fabric.</text>
<defs><marker id="ae4" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>RD and RT. The RD keeps addresses unique across tenants; the RT decides who imports the route. Together they give multi-tenant isolation on one fabric.</figcaption>
</figure>

Many implementations can derive both the RD and the RT automatically from the VNI and the local ASN, so in practice you often just assign a VNI and the control plane fills in sensible RD and RT values for you.

## What EVPN buys us

With this machinery in place, the Chapter 6 complaints dissolve:

- **No discovery flooding.** Every VTEP already holds every remote host, learned from BGP, not from traffic.
- **Proactive and fast.** A new or moved host is advertised the instant it appears, so there is no first-packet flood and no waiting for a stale entry to age out.
- **ARP suppression becomes possible.** Because a VTEP knows the IP-to-MAC mapping of remote hosts from EVPN, it can answer a local ARP request itself instead of flooding it across the fabric. Chapter 9 shows this in action.
- **Scales on proven technology.** It is all just BGP, the same protocol the internet runs on, with route reflectors to keep sessions manageable.

To actually carry all this information, EVPN defines several **route types**, each a different kind of advertisement for a different purpose. Learning those five route types is the subject of the next chapter.
