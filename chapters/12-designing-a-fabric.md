# Chapter 12: Designing a Fabric

The concepts are complete. This chapter turns them into design decisions: how big to make the fabric, how to number everything, where routing happens, and how to keep the whole thing consistent. The goal is a design you could hand to an automation tool and build.

## Sizing: ports decide everything

The size of a single spine-leaf fabric is set by **port counts**. Two numbers dominate:

- The number of **uplink ports on a leaf** sets the maximum number of spines, because a leaf connects once to each spine.
- The number of **ports on a spine** sets the maximum number of leaves, because a spine connects once to each leaf.

<figure class="diagram">
<svg viewBox="0 0 680 220" style="width:100%;height:auto;max-width:680px" role="img" aria-label="A leaf with some uplink ports and some server ports, showing that uplink count caps the spine count and the downlink to uplink ratio sets oversubscription." xmlns="http://www.w3.org/2000/svg">
<style>
.d1{ font-family:var(--sans),sans-serif; }
.d1 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.d1 .up{ fill:var(--accent); stroke:var(--accent); stroke-width:1; }
.d1 .down{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1; }
.d1 .s{ fill:var(--text-dim); font-size:13px; }
.d1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.d1 .calc{ fill:var(--accent); font-size:13px; font-family:var(--mono),monospace; }
</style>
<g class="d1">
<rect x="180" y="70" width="320" height="40" rx="6" class="leaf"/><text x="340" y="94" text-anchor="middle" class="s">leaf switch</text>
<rect x="200" y="40" width="16" height="16" rx="3" class="up"/><rect x="224" y="40" width="16" height="16" rx="3" class="up"/><rect x="248" y="40" width="16" height="16" rx="3" class="up"/><rect x="272" y="40" width="16" height="16" rx="3" class="up"/>
<text x="245" y="30" class="lbl">uplinks to spines (caps spine count)</text>
<rect x="200" y="124" width="14" height="14" rx="3" class="down"/><rect x="220" y="124" width="14" height="14" rx="3" class="down"/><rect x="240" y="124" width="14" height="14" rx="3" class="down"/><rect x="260" y="124" width="14" height="14" rx="3" class="down"/><rect x="280" y="124" width="14" height="14" rx="3" class="down"/><rect x="300" y="124" width="14" height="14" rx="3" class="down"/><rect x="320" y="124" width="14" height="14" rx="3" class="down"/><rect x="340" y="124" width="14" height="14" rx="3" class="down"/><rect x="360" y="124" width="14" height="14" rx="3" class="down"/><rect x="380" y="124" width="14" height="14" rx="3" class="down"/><rect x="400" y="124" width="14" height="14" rx="3" class="down"/><rect x="420" y="124" width="14" height="14" rx="3" class="down"/>
<text x="300" y="158" class="lbl">server ports</text>
<text x="340" y="196" text-anchor="middle" class="calc">oversubscription = total server bandwidth / total uplink bandwidth</text>
</g>
</svg>
<figcaption>Leaf port math. Uplinks cap the spine count; the ratio of server bandwidth to uplink bandwidth is the oversubscription from Chapter 1.</figcaption>
</figure>

A worked example. Say a leaf has 48 server ports at 25 Gbps and 8 uplink ports at 100 Gbps. That is 1200 Gbps of server capacity against 800 Gbps of uplink, an oversubscription of 1.5:1, which is healthy for most workloads. The 8 uplinks also mean the fabric can have up to 8 spines. If each spine has 64 ports, the fabric can hold up to 64 leaves. Multiply it out and a single two-tier fabric here supports 64 leaves times 48 servers, roughly 3000 servers, which covers a large room.

## Growing past one fabric: pods and super-spines

When one fabric is not enough, you do not build a bigger spine. You build **more fabrics** and connect them with another CLOS stage. Each spine-leaf fabric becomes a **pod**, and a layer of **super-spine** switches connects the pods. This is a 5-stage (or 3-tier) CLOS, and it is how hyperscale data centers reach tens of thousands of servers while still using the same small fixed switches.

<figure class="diagram">
<svg viewBox="0 0 700 250" style="width:100%;height:auto;max-width:700px" role="img" aria-label="Two pods, each a spine-leaf fabric, connected by a layer of super-spine switches above them." xmlns="http://www.w3.org/2000/svg">
<style>
.d2{ font-family:var(--sans),sans-serif; }
.d2 .ss{ fill:var(--accent); stroke:var(--accent); stroke-width:1.4; }
.d2 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.4; }
.d2 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.d2 .pod{ fill:none; stroke:var(--border); stroke-width:1.4; stroke-dasharray:5 4; }
.d2 .ln{ stroke:var(--border); stroke-width:1.1; fill:none; }
.d2 .ssln{ stroke:var(--accent); stroke-width:1.3; fill:none; }
.d2 .s{ fill:var(--text-dim); font-size:13px; }
.d2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="d2">
<rect x="250" y="20" width="80" height="24" rx="5" class="ss"/><text x="290" y="37" text-anchor="middle" class="s" style="fill:var(--bg)">S-Spine 1</text>
<rect x="370" y="20" width="80" height="24" rx="5" class="ss"/><text x="410" y="37" text-anchor="middle" class="s" style="fill:var(--bg)">S-Spine 2</text>
<rect x="70" y="150" width="230" height="80" rx="8" class="pod"/><text x="90" y="170" class="lbl">Pod 1</text>
<rect x="400" y="150" width="230" height="80" rx="8" class="pod"/><text x="420" y="170" class="lbl">Pod 2</text>
<rect x="110" y="110" width="70" height="22" rx="5" class="spine"/><text x="145" y="126" text-anchor="middle" class="s">spine</text>
<rect x="200" y="110" width="70" height="22" rx="5" class="spine"/><text x="235" y="126" text-anchor="middle" class="s">spine</text>
<rect x="440" y="110" width="70" height="22" rx="5" class="spine"/><text x="475" y="126" text-anchor="middle" class="s">spine</text>
<rect x="530" y="110" width="70" height="22" rx="5" class="spine"/><text x="565" y="126" text-anchor="middle" class="s">spine</text>
<rect x="100" y="188" width="60" height="20" rx="4" class="leaf"/><rect x="170" y="188" width="60" height="20" rx="4" class="leaf"/><rect x="430" y="188" width="60" height="20" rx="4" class="leaf"/><rect x="500" y="188" width="60" height="20" rx="4" class="leaf"/>
<line class="ln" x1="145" y1="132" x2="130" y2="188"/><line class="ln" x1="145" y1="132" x2="200" y2="188"/><line class="ln" x1="235" y1="132" x2="130" y2="188"/><line class="ln" x1="235" y1="132" x2="200" y2="188"/>
<line class="ln" x1="475" y1="132" x2="460" y2="188"/><line class="ln" x1="475" y1="132" x2="530" y2="188"/><line class="ln" x1="565" y1="132" x2="460" y2="188"/><line class="ln" x1="565" y1="132" x2="530" y2="188"/>
<line class="ssln" x1="290" y1="44" x2="145" y2="110"/><line class="ssln" x1="290" y1="44" x2="235" y2="110"/><line class="ssln" x1="410" y1="44" x2="145" y2="110"/><line class="ssln" x1="410" y1="44" x2="235" y2="110"/><line class="ssln" x1="290" y1="44" x2="475" y2="110"/><line class="ssln" x1="410" y1="44" x2="565" y2="110"/><line class="ssln" x1="290" y1="44" x2="565" y2="110"/><line class="ssln" x1="410" y1="44" x2="475" y2="110"/>
<text x="350" y="246" text-anchor="middle" class="lbl">Same CLOS idea, one level up. Pods scale out just as leaves did inside a pod.</text>
</g>
</svg>
<figcaption>A multi-pod fabric. Super-spines connect pods exactly as spines connect leaves, so the design scales recursively.</figcaption>
</figure>

## A numbering plan

A fabric lives or dies by consistent addressing, because automation depends on it. Decide these ranges up front and never improvise them per switch:

<figure class="diagram">
<svg viewBox="0 0 680 232" style="width:100%;height:auto;max-width:680px" role="img" aria-label="A table of numbering plan ranges: loopbacks, point to point links, leaf and spine ASNs, L2VNIs per subnet, and L3VNIs per tenant." xmlns="http://www.w3.org/2000/svg">
<style>
.d3{ font-family:var(--sans),sans-serif; }
.d3 .row{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.2; }
.d3 .rowalt{ fill:var(--panel); stroke:var(--border); stroke-width:1.2; }
.d3 .hdr{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.4; }
.d3 .k{ fill:var(--text); font-size:13px; font-weight:700; }
.d3 .v{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.d3 .h{ fill:var(--text); font-size:15px; font-weight:700; }
</style>
<g class="d3">
<rect x="20" y="16" width="640" height="28" rx="5" class="hdr"/><text x="40" y="35" class="h">What</text><text x="300" y="35" class="h">Example range</text>
<rect x="20" y="48" width="640" height="26" rx="4" class="row"/><text x="40" y="65" class="k">Loopbacks (VTEP IPs)</text><text x="300" y="65" class="v">10.0.0.0/24 spines, 10.0.1.0/24 leaves</text>
<rect x="20" y="78" width="640" height="26" rx="4" class="rowalt"/><text x="40" y="95" class="k">Point-to-point links</text><text x="300" y="95" class="v">10.1.0.0/16 as /31s, or BGP unnumbered</text>
<rect x="20" y="108" width="640" height="26" rx="4" class="row"/><text x="40" y="125" class="k">Spine ASN</text><text x="300" y="125" class="v">65000 (shared)</text>
<rect x="20" y="138" width="640" height="26" rx="4" class="rowalt"/><text x="40" y="155" class="k">Leaf ASNs</text><text x="300" y="155" class="v">65101, 65102, 65103, ... (unique)</text>
<rect x="20" y="168" width="640" height="26" rx="4" class="row"/><text x="40" y="185" class="k">L2VNI per subnet</text><text x="300" y="185" class="v">10010, 10020, ... (one per subnet)</text>
<rect x="20" y="198" width="640" height="26" rx="4" class="rowalt"/><text x="40" y="215" class="k">L3VNI per tenant VRF</text><text x="300" y="215" class="v">10999, 10998, ... (one per VRF)</text>
</g>
</svg>
<figcaption>A sample numbering plan. The exact numbers matter less than picking ranges once and applying them by rule everywhere.</figcaption>
</figure>

A common trick is to **encode position into addresses**. If Leaf 7's loopback is `10.0.1.7` and its ASN is `65107`, a human or a script can read a switch's identity straight from its numbers. That regularity is what makes the fabric automatable.

## Where to put routing and borders

Two placement decisions shape the design:

- **Routing at the leaf, always.** Use the distributed anycast gateway from Chapter 10 so every leaf routes for its local subnets. Avoid centralized gateways except where an external constraint forces one.
- **Dedicated border leaves.** Pick a pair of leaves to be **border leaves** that connect the fabric to the outside: the WAN, the internet edge, firewalls, and load balancers. They advertise external destinations into the fabric as type 5 routes and summarize fabric subnets outward. Keeping this role on dedicated leaves keeps the rest of the fabric uniform.

<figure class="diagram">
<svg viewBox="0 0 680 236" style="width:100%;height:auto;max-width:680px" role="img" aria-label="A fabric with ordinary server leaves and a pair of border leaves that connect out to the WAN and external services." xmlns="http://www.w3.org/2000/svg">
<style>
.d4{ font-family:var(--sans),sans-serif; }
.d4 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.4; }
.d4 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.d4 .border{ fill:var(--accent); stroke:var(--accent); stroke-width:1.4; }
.d4 .ext{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.4; }
.d4 .ln{ stroke:var(--border); stroke-width:1.2; fill:none; }
.d4 .s{ fill:var(--text-dim); font-size:13px; }
.d4 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="d4">
<rect x="230" y="30" width="90" height="24" rx="5" class="spine"/><text x="275" y="47" text-anchor="middle" class="s">Spine 1</text>
<rect x="360" y="30" width="90" height="24" rx="5" class="spine"/><text x="405" y="47" text-anchor="middle" class="s">Spine 2</text>
<rect x="60" y="110" width="90" height="26" rx="5" class="leaf"/><text x="105" y="127" text-anchor="middle" class="s">server leaf</text>
<rect x="180" y="110" width="90" height="26" rx="5" class="leaf"/><text x="225" y="127" text-anchor="middle" class="s">server leaf</text>
<rect x="420" y="110" width="90" height="26" rx="5" class="border"/><text x="465" y="127" text-anchor="middle" class="s" style="fill:var(--bg)">border leaf</text>
<rect x="540" y="110" width="90" height="26" rx="5" class="border"/><text x="585" y="127" text-anchor="middle" class="s" style="fill:var(--bg)">border leaf</text>
<line class="ln" x1="275" y1="54" x2="105" y2="110"/><line class="ln" x1="275" y1="54" x2="225" y2="110"/><line class="ln" x1="275" y1="54" x2="465" y2="110"/><line class="ln" x1="275" y1="54" x2="585" y2="110"/>
<line class="ln" x1="405" y1="54" x2="105" y2="110"/><line class="ln" x1="405" y1="54" x2="225" y2="110"/><line class="ln" x1="405" y1="54" x2="465" y2="110"/><line class="ln" x1="405" y1="54" x2="585" y2="110"/>
<rect x="430" y="168" width="240" height="28" rx="5" class="ext"/><text x="550" y="187" text-anchor="middle" class="s">WAN / internet / firewalls</text>
<line class="ln" x1="465" y1="136" x2="500" y2="168"/><line class="ln" x1="585" y1="136" x2="600" y2="168"/>
<text x="340" y="224" text-anchor="middle" class="lbl">Server leaves stay identical; the border role is isolated to a dedicated pair.</text>
</g>
</svg>
<figcaption>Border leaves. Isolating the external-connectivity role keeps every server leaf uniform and easy to template.</figcaption>
</figure>

## Build it with automation, not by hand

A fabric of dozens of identical switches should never be configured one device at a time. The uniformity we have designed in is precisely what lets a template plus a small data file per switch generate every configuration. Tools like Ansible, or vendor fabric managers, render the templates and push them. The payoff is consistency: every leaf is configured by the same rule, so a whole class of human error disappears.

<figure class="diagram">
<svg viewBox="0 0 680 180" style="width:100%;height:auto;max-width:680px" role="img" aria-label="A configuration template plus a per-switch data file are rendered by automation into full configs pushed to every switch." xmlns="http://www.w3.org/2000/svg">
<style>
.d5{ font-family:var(--sans),sans-serif; }
.d5 .in{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.d5 .eng{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.d5 .out{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.d5 .fl{ stroke:var(--accent); stroke-width:1.8; fill:none; }
.d5 .s{ fill:var(--text-dim); font-size:13px; }
.d5 .t{ fill:var(--text); font-size:13px; font-weight:700; }
.d5 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="d5">
<rect x="30" y="40" width="130" height="34" rx="6" class="in"/><text x="95" y="62" text-anchor="middle" class="s">template</text>
<rect x="30" y="94" width="130" height="34" rx="6" class="in"/><text x="95" y="116" text-anchor="middle" class="s">per-switch data</text>
<rect x="270" y="66" width="140" height="40" rx="8" class="eng"/><text x="340" y="90" text-anchor="middle" class="t">automation</text>
<rect x="520" y="34" width="130" height="24" rx="5" class="out"/><text x="585" y="51" text-anchor="middle" class="s">leaf-1 config</text>
<rect x="520" y="66" width="130" height="24" rx="5" class="out"/><text x="585" y="83" text-anchor="middle" class="s">leaf-2 config</text>
<rect x="520" y="98" width="130" height="24" rx="5" class="out"/><text x="585" y="115" text-anchor="middle" class="s">spine-1 config ...</text>
<line class="fl" x1="160" y1="57" x2="270" y2="80" marker-end="url(#ad5)"/><line class="fl" x1="160" y1="111" x2="270" y2="92" marker-end="url(#ad5)"/><line class="fl" x1="410" y1="80" x2="520" y2="46" marker-end="url(#ad5)"/><line class="fl" x1="410" y1="86" x2="520" y2="78" marker-end="url(#ad5)"/><line class="fl" x1="410" y1="92" x2="520" y2="110" marker-end="url(#ad5)"/>
<text x="340" y="150" text-anchor="middle" class="lbl">One rule, many switches. Uniformity is what makes this possible, so design for it.</text>
<defs><marker id="ad5" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Templated configuration. The regularity designed into the fabric is what lets one template plus small data files build every switch identically.</figcaption>
</figure>

With a design in hand and built, the job shifts to keeping it healthy. The final chapter covers how to operate and troubleshoot a live fabric.
