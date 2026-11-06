# Chapter 2: The Spine-Leaf CLOS Fabric

The answer to the problems in Chapter 1 is a topology called **spine-leaf**, a two-stage folded **CLOS fabric**. The idea comes from Charles Clos, who in 1952 designed multi-stage telephone switching networks that could carry any call without blocking, built from small, cheap switches instead of one impossibly large one. The same math works for packets.

## The shape of the fabric

A spine-leaf fabric has exactly two kinds of switch. **Leaf** switches sit at the edge and connect to servers. **Spine** switches sit in the middle and connect only to leaves. The defining rule is simple:

> Every leaf connects to every spine. No leaf connects to another leaf, and no spine connects to another spine.

<figure class="diagram">
<svg viewBox="0 0 700 300" style="width:100%;height:auto;max-width:700px" role="img" aria-label="Four leaf switches each connected to all three spine switches, with servers hanging off the leaves." xmlns="http://www.w3.org/2000/svg">
<style>
.f1{ font-family:var(--sans),sans-serif; }
.f1 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.f1 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.f1 .srv{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1; }
.f1 .ln{ stroke:var(--border); stroke-width:1.3; fill:none; }
.f1 .h{ fill:var(--text); font-size:15px; font-weight:700; }
.f1 .s{ fill:var(--text-dim); font-size:13px; }
.f1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="f1">
<line class="ln" x1="200" y1="56" x2="110" y2="150"/><line class="ln" x1="200" y1="56" x2="290" y2="150"/><line class="ln" x1="200" y1="56" x2="470" y2="150"/><line class="ln" x1="200" y1="56" x2="650" y2="150"/>
<line class="ln" x1="380" y1="56" x2="110" y2="150"/><line class="ln" x1="380" y1="56" x2="290" y2="150"/><line class="ln" x1="380" y1="56" x2="470" y2="150"/><line class="ln" x1="380" y1="56" x2="650" y2="150"/>
<line class="ln" x1="560" y1="56" x2="110" y2="150"/><line class="ln" x1="560" y1="56" x2="290" y2="150"/><line class="ln" x1="560" y1="56" x2="470" y2="150"/><line class="ln" x1="560" y1="56" x2="650" y2="150"/>
<rect x="150" y="32" width="100" height="28" rx="6" class="spine"/><text x="200" y="51" text-anchor="middle" class="s">Spine 1</text>
<rect x="330" y="32" width="100" height="28" rx="6" class="spine"/><text x="380" y="51" text-anchor="middle" class="s">Spine 2</text>
<rect x="510" y="32" width="100" height="28" rx="6" class="spine"/><text x="560" y="51" text-anchor="middle" class="s">Spine 3</text>
<rect x="66" y="150" width="88" height="26" rx="6" class="leaf"/><text x="110" y="168" text-anchor="middle" class="s">Leaf 1</text>
<rect x="246" y="150" width="88" height="26" rx="6" class="leaf"/><text x="290" y="168" text-anchor="middle" class="s">Leaf 2</text>
<rect x="426" y="150" width="88" height="26" rx="6" class="leaf"/><text x="470" y="168" text-anchor="middle" class="s">Leaf 3</text>
<rect x="606" y="150" width="88" height="26" rx="6" class="leaf"/><text x="650" y="168" text-anchor="middle" class="s">Leaf 4</text>
<rect x="74" y="200" width="20" height="18" rx="3" class="srv"/><rect x="100" y="200" width="20" height="18" rx="3" class="srv"/><rect x="126" y="200" width="20" height="18" rx="3" class="srv"/>
<rect x="254" y="200" width="20" height="18" rx="3" class="srv"/><rect x="280" y="200" width="20" height="18" rx="3" class="srv"/><rect x="306" y="200" width="20" height="18" rx="3" class="srv"/>
<rect x="434" y="200" width="20" height="18" rx="3" class="srv"/><rect x="460" y="200" width="20" height="18" rx="3" class="srv"/><rect x="486" y="200" width="20" height="18" rx="3" class="srv"/>
<rect x="614" y="200" width="20" height="18" rx="3" class="srv"/><rect x="640" y="200" width="20" height="18" rx="3" class="srv"/><rect x="666" y="200" width="20" height="18" rx="3" class="srv"/>
<line class="ln" x1="110" y1="176" x2="110" y2="200"/><line class="ln" x1="290" y1="176" x2="290" y2="200"/><line class="ln" x1="470" y1="176" x2="470" y2="200"/><line class="ln" x1="650" y1="176" x2="650" y2="200"/>
<text x="350" y="260" text-anchor="middle" class="h">Every leaf reaches every other leaf through any spine, in exactly two hops.</text>
<text x="350" y="282" text-anchor="middle" class="lbl">Spines are transit only. Leaves are where servers connect.</text>
</g>
</svg>
<figcaption>A spine-leaf fabric. The full mesh between the two layers is the whole design.</figcaption>
</figure>

## Equal cost, every path

Because every leaf connects to every spine, any two leaves are joined by as many parallel paths as there are spines. In the figure above, Leaf 1 can reach Leaf 4 through Spine 1, Spine 2, or Spine 3. All three paths are the same length and the same cost, so the fabric uses **Equal-Cost Multi-Path (ECMP)** to spread traffic across all of them at once.

<figure class="diagram">
<svg viewBox="0 0 680 230" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Traffic from Leaf 1 to Leaf 4 is split across three spines by ECMP, using all paths at once." xmlns="http://www.w3.org/2000/svg">
<style>
.f2{ font-family:var(--sans),sans-serif; }
.f2 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.f2 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.f2 .act{ stroke:var(--accent); stroke-width:2.6; fill:none; }
.f2 .idle{ stroke:var(--border); stroke-width:1.2; fill:none; }
.f2 .s{ fill:var(--text-dim); font-size:13px; }
.f2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.f2 .h{ fill:var(--text); font-size:15px; font-weight:700; }
</style>
<g class="f2">
<line class="act" x1="110" y1="150" x2="200" y2="56"/><line class="act" x1="110" y1="150" x2="340" y2="56"/><line class="act" x1="110" y1="150" x2="480" y2="56"/>
<line class="act" x1="200" y1="56" x2="600" y2="150"/><line class="act" x1="340" y1="56" x2="600" y2="150"/><line class="act" x1="480" y1="56" x2="600" y2="150"/>
<rect x="150" y="32" width="100" height="26" rx="6" class="spine"/><text x="200" y="50" text-anchor="middle" class="s">Spine 1</text>
<rect x="290" y="32" width="100" height="26" rx="6" class="spine"/><text x="340" y="50" text-anchor="middle" class="s">Spine 2</text>
<rect x="430" y="32" width="100" height="26" rx="6" class="spine"/><text x="480" y="50" text-anchor="middle" class="s">Spine 3</text>
<rect x="66" y="148" width="88" height="26" rx="6" class="leaf"/><text x="110" y="166" text-anchor="middle" class="s">Leaf 1</text>
<rect x="556" y="148" width="88" height="26" rx="6" class="leaf"/><text x="600" y="166" text-anchor="middle" class="s">Leaf 4</text>
<text x="350" y="200" text-anchor="middle" class="h">One flow, three equal paths, all carrying traffic.</text>
<text x="350" y="220" text-anchor="middle" class="lbl">Add a spine and every leaf pair gains another path. Nothing is blocked.</text>
</g>
</svg>
<figcaption>ECMP uses all the parallel spines at once. This is the opposite of spanning tree, which left redundant links idle.</figcaption>
</figure>

This directly solves two problems from Chapter 1. Every link forwards traffic, so no capacity is wasted. And because the paths are equal, latency between any two servers is consistent: the traffic always crosses one leaf, one spine, and one more leaf. Engineers call this a **non-blocking** or predictable fabric when the spine capacity is sized to match the leaf uplinks.

## Scale-out, not scale-up

The old core grew by **scale-up**: when it ran out of capacity, you replaced it with a bigger, costlier chassis. A spine-leaf fabric grows by **scale-out** instead.

- **Need more server ports?** Add a leaf. It connects to the existing spines and is immediately reachable.
- **Need more bandwidth between leaves?** Add a spine. Every leaf gains one more parallel path, so the whole fabric gets wider at once.

<figure class="diagram">
<svg viewBox="0 0 680 210" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Adding a fourth spine adds a link to every existing leaf, widening the whole fabric." xmlns="http://www.w3.org/2000/svg">
<style>
.f3{ font-family:var(--sans),sans-serif; }
.f3 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.f3 .new{ fill:var(--bg-soft); stroke:var(--accent); stroke-width:1.8; stroke-dasharray:5 4; }
.f3 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.f3 .ln{ stroke:var(--border); stroke-width:1.2; fill:none; }
.f3 .nln{ stroke:var(--accent); stroke-width:1.6; stroke-dasharray:5 4; fill:none; }
.f3 .s{ fill:var(--text-dim); font-size:13px; }
.f3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="f3">
<line class="ln" x1="150" y1="56" x2="120" y2="140"/><line class="ln" x1="150" y1="56" x2="300" y2="140"/><line class="ln" x1="150" y1="56" x2="480" y2="140"/>
<line class="ln" x1="300" y1="56" x2="120" y2="140"/><line class="ln" x1="300" y1="56" x2="300" y2="140"/><line class="ln" x1="300" y1="56" x2="480" y2="140"/>
<line class="ln" x1="450" y1="56" x2="120" y2="140"/><line class="ln" x1="450" y1="56" x2="300" y2="140"/><line class="ln" x1="450" y1="56" x2="480" y2="140"/>
<line class="nln" x1="600" y1="56" x2="120" y2="140"/><line class="nln" x1="600" y1="56" x2="300" y2="140"/><line class="nln" x1="600" y1="56" x2="480" y2="140"/>
<rect x="105" y="34" width="90" height="24" rx="6" class="spine"/><text x="150" y="51" text-anchor="middle" class="s">Spine 1</text>
<rect x="255" y="34" width="90" height="24" rx="6" class="spine"/><text x="300" y="51" text-anchor="middle" class="s">Spine 2</text>
<rect x="405" y="34" width="90" height="24" rx="6" class="spine"/><text x="450" y="51" text-anchor="middle" class="s">Spine 3</text>
<rect x="555" y="34" width="90" height="24" rx="6" class="new"/><text x="600" y="51" text-anchor="middle" class="s">Spine 4</text>
<rect x="80" y="140" width="80" height="24" rx="6" class="leaf"/><text x="120" y="157" text-anchor="middle" class="s">Leaf 1</text>
<rect x="260" y="140" width="80" height="24" rx="6" class="leaf"/><text x="300" y="157" text-anchor="middle" class="s">Leaf 2</text>
<rect x="440" y="140" width="80" height="24" rx="6" class="leaf"/><text x="480" y="157" text-anchor="middle" class="s">Leaf 3</text>
<text x="340" y="190" text-anchor="middle" class="lbl">The dashed spine and its links are the new capacity. The rest of the fabric is untouched.</text>
</g>
</svg>
<figcaption>Scale-out growth. Adding one spine adds a path to every leaf, so capacity grows smoothly and cheaply.</figcaption>
</figure>

A practical ceiling exists: a leaf can only have as many spines as it has uplink ports, and a spine can only reach as many leaves as it has ports. When a single fabric is not enough, operators build a larger CLOS by adding a third stage, often called a **super-spine** or a **5-stage** fabric, which connects several spine-leaf **pods** together. The principle is the same at every level.

## Fixed switches, uniform fabric

Spine-leaf also changed the hardware. Instead of a few giant modular chassis, a fabric is built from many identical **fixed-form** switches, typically 1 or 2 rack units, often running merchant silicon from vendors like Broadcom. Every spine is the same model; every leaf is the same model. This uniformity makes the fabric cheaper to buy, simpler to stock as spares, and far easier to automate, because one configuration template covers every switch of a given role.

## What the fabric still needs

We now have a topology where every leaf can reach every other leaf over many equal paths. But a topology is just cabling. For packets to actually flow, the switches must agree on how to reach each other, which means they need a routing protocol running across these links. That routing layer is called the **underlay**, and it is the subject of the next chapter.
