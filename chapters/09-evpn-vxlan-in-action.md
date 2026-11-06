# Chapter 9: EVPN-VXLAN in Action

Theory is in place. This chapter runs a real sequence of events through an EVPN-VXLAN fabric: a host comes online, its location spreads through BGP, another host resolves it without any fabric flooding, and the two exchange traffic. We also look at the configuration that makes a leaf behave this way.

## The scenario

Two servers in the same subnet and the same segment, VNI 10, sit under different leaves. We will bring VM A online and watch VM B reach it.

<figure class="diagram">
<svg viewBox="0 0 680 230" style="width:100%;height:auto;max-width:680px" role="img" aria-label="VM A under Leaf 1 and VM B under Leaf 3, both in VNI 10, with spine route reflectors between them." xmlns="http://www.w3.org/2000/svg">
<style>
.a0{ font-family:var(--sans),sans-serif; }
.a0 .spine{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.a0 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.a0 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.a0 .ln{ stroke:var(--border); stroke-width:1.3; fill:none; }
.a0 .s{ fill:var(--text-dim); font-size:13px; }
.a0 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
</style>
<g class="a0">
<line class="ln" x1="250" y1="52" x2="130" y2="110"/><line class="ln" x1="250" y1="52" x2="550" y2="110"/><line class="ln" x1="430" y1="52" x2="130" y2="110"/><line class="ln" x1="430" y1="52" x2="550" y2="110"/>
<rect x="200" y="28" width="100" height="26" rx="6" class="spine"/><text x="250" y="46" text-anchor="middle" class="s">Spine 1 (RR)</text>
<rect x="380" y="28" width="100" height="26" rx="6" class="spine"/><text x="430" y="46" text-anchor="middle" class="s">Spine 2 (RR)</text>
<rect x="70" y="110" width="120" height="44" rx="6" class="leaf"/><text x="130" y="129" text-anchor="middle" class="s">Leaf 1</text><text x="130" y="146" text-anchor="middle" class="mono">lo 10.0.1.1</text>
<rect x="490" y="110" width="120" height="44" rx="6" class="leaf"/><text x="550" y="129" text-anchor="middle" class="s">Leaf 3</text><text x="550" y="146" text-anchor="middle" class="mono">lo 10.0.1.3</text>
<rect x="75" y="172" width="110" height="44" rx="6" class="node"/><text x="130" y="190" text-anchor="middle" class="s">VM A</text><text x="130" y="207" text-anchor="middle" class="mono">10.1.1.5</text>
<rect x="495" y="172" width="110" height="44" rx="6" class="node"/><text x="550" y="190" text-anchor="middle" class="s">VM B</text><text x="550" y="207" text-anchor="middle" class="mono">10.1.1.6</text>
<line class="ln" x1="130" y1="154" x2="130" y2="172"/><line class="ln" x1="550" y1="154" x2="550" y2="172"/>
</g>
</svg>
<figcaption>The scenario. VM A and VM B share subnet 10.1.1.0/24 and VNI 10 but sit under Leaf 1 and Leaf 3, with the spines acting as route reflectors.</figcaption>
</figure>

## Step 1: VM A appears, Leaf 1 advertises it

When VM A sends its first frame, or its gratuitous ARP on boot, Leaf 1 learns VM A locally: MAC `...05`, IP `10.1.1.5`, in VNI 10. Leaf 1 immediately builds a **type 2 route** for it and sends it to the spines, which reflect it to every other leaf.

<figure class="diagram">
<svg viewBox="0 0 680 200" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Leaf 1 learns VM A locally and advertises a type 2 route through the spine route reflector to Leaf 3, which installs VM A as reachable via Leaf 1." xmlns="http://www.w3.org/2000/svg">
<style>
.a1{ font-family:var(--sans),sans-serif; }
.a1 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.a1 .rr{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.a1 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.a1 .adv{ stroke:var(--accent); stroke-width:2; fill:none; }
.a1 .loc{ stroke:var(--text-faint); stroke-width:1.6; fill:none; }
.a1 .s{ fill:var(--text-dim); font-size:13px; }
.a1 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.a1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.a1 .step{ fill:var(--text); font-size:13px; font-weight:700; }
</style>
<g class="a1">
<rect x="30" y="92" width="56" height="26" rx="5" class="node"/><text x="58" y="109" text-anchor="middle" class="s">VM A</text>
<rect x="110" y="84" width="84" height="42" rx="6" class="leaf"/><text x="152" y="109" text-anchor="middle" class="s">Leaf 1</text>
<line class="loc" x1="86" y1="105" x2="110" y2="105" marker-end="url(#aa1)"/><text x="152" y="150" text-anchor="middle" class="step">learn local</text>
<rect x="320" y="86" width="90" height="38" rx="6" class="rr"/><text x="365" y="110" text-anchor="middle" class="s">Spine (RR)</text>
<line class="adv" x1="194" y1="105" x2="320" y2="105" marker-end="url(#aa1)"/>
<rect x="540" y="84" width="90" height="42" rx="6" class="leaf"/><text x="585" y="109" text-anchor="middle" class="s">Leaf 3</text>
<line class="adv" x1="410" y1="105" x2="540" y2="105" marker-end="url(#aa1)"/>
<text x="365" y="66" text-anchor="middle" class="mono">type 2: MAC ...05, IP .5, VNI 10, via 10.0.1.1</text>
<text x="585" y="150" text-anchor="middle" class="mono">installs: VM A via Leaf 1</text>
<text x="340" y="182" text-anchor="middle" class="lbl">Leaf 3 now knows VM A before VM B has sent anything to it.</text>
<defs><marker id="aa1" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Step 1. VM A is learned locally and advertised as a type 2 route. Leaf 3 installs it immediately, with no traffic from VM A to VM B yet.</figcaption>
</figure>

## Step 2: VM B resolves VM A, locally

Now VM B wants to talk to `10.1.1.5`. First it must resolve that IP to a MAC with ARP, which is normally a broadcast. But Leaf 3 already holds the type 2 route carrying both VM A's IP and MAC. So Leaf 3 performs **ARP suppression**: it answers VM B's ARP request itself, out of its EVPN knowledge, and the broadcast never touches the fabric.

<figure class="diagram">
<svg viewBox="0 0 680 210" style="width:100%;height:auto;max-width:680px" role="img" aria-label="VM B broadcasts an ARP for VM A's IP, and Leaf 3 answers it locally from EVPN state instead of flooding it across the fabric." xmlns="http://www.w3.org/2000/svg">
<style>
.a2{ font-family:var(--sans),sans-serif; }
.a2 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.a2 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.a2 .fab{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.a2 .q{ stroke:var(--text-faint); stroke-width:1.8; fill:none; }
.a2 .ans{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.a2 .blk{ stroke:var(--text-faint); stroke-width:1.8; fill:none; stroke-dasharray:4 4; }
.a2 .s{ fill:var(--text-dim); font-size:13px; }
.a2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.a2 .mono{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.a2 .no{ fill:var(--text-faint); font-size:13px; font-weight:700; }
</style>
<g class="a2">
<rect x="20" y="90" width="66" height="28" rx="6" class="node"/><text x="53" y="108" text-anchor="middle" class="s">VM B</text>
<rect x="160" y="80" width="170" height="48" rx="6" class="leaf"/><text x="245" y="100" text-anchor="middle" class="s">Leaf 3</text><text x="245" y="117" text-anchor="middle" class="mono">knows VM A from EVPN</text>
<line class="q" x1="94" y1="100" x2="160" y2="100" marker-end="url(#aa2)"/><text x="127" y="88" text-anchor="middle" class="lbl">ARP?</text>
<line class="ans" x1="160" y1="112" x2="94" y2="112" marker-end="url(#aa2b)"/><text x="127" y="136" text-anchor="middle" class="mono">MAC ...05</text>
<rect x="480" y="86" width="160" height="40" rx="6" class="fab"/><text x="560" y="110" text-anchor="middle" class="s">rest of the fabric</text>
<line class="blk" x1="330" y1="104" x2="480" y2="104"/><text x="405" y="92" text-anchor="middle" class="no">no broadcast sent</text>
<text x="350" y="180" text-anchor="middle" class="lbl">The ARP is answered at the edge. BUM flooding that plagued Chapter 6 simply does not happen.</text>
<defs><marker id="aa2" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--text-faint)"/></marker><marker id="aa2b" markerWidth="8" markerHeight="8" refX="2" refY="4" orient="auto"><path d="M8 0 L0 4 L8 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Step 2. ARP suppression. Leaf 3 answers VM B's ARP from its EVPN table, so the request never floods the fabric. This is a direct payoff of the type 2 IP field.</figcaption>
</figure>

## Step 3: the data packet crosses the fabric

VM B now has VM A's MAC and sends a unicast frame. Leaf 3 looks up that MAC, finds from the type 2 route that it lives behind Leaf 1, and encapsulates exactly as in Chapter 5: VXLAN with VNI 10, outer IP from Leaf 3's loopback to Leaf 1's loopback. The fabric ECMP-routes it to Leaf 1, which decapsulates and delivers it to VM A.

<figure class="diagram">
<svg viewBox="0 0 700 180" style="width:100%;height:auto;max-width:700px" role="img" aria-label="VM B's frame is encapsulated by Leaf 3 to Leaf 1's loopback, routed across a spine, and decapsulated to reach VM A, all as known unicast with no flooding." xmlns="http://www.w3.org/2000/svg">
<style>
.a3{ font-family:var(--sans),sans-serif; }
.a3 .leaf{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.a3 .node{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.3; }
.a3 .spine{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.a3 .fl{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.a3 .s{ fill:var(--text-dim); font-size:13px; }
.a3 .mono{ fill:var(--text-faint); font-size:12.5px; font-family:var(--mono),monospace; }
.a3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="a3">
<rect x="20" y="70" width="60" height="28" rx="6" class="node"/><text x="50" y="88" text-anchor="middle" class="s">VM B</text>
<rect x="110" y="64" width="84" height="40" rx="6" class="leaf"/><text x="152" y="88" text-anchor="middle" class="s">Leaf 3</text>
<rect x="320" y="68" width="80" height="32" rx="6" class="spine"/><text x="360" y="88" text-anchor="middle" class="s">Spine</text>
<rect x="520" y="64" width="84" height="40" rx="6" class="leaf"/><text x="562" y="88" text-anchor="middle" class="s">Leaf 1</text>
<rect x="636" y="70" width="56" height="28" rx="6" class="node"/><text x="664" y="88" text-anchor="middle" class="s">VM A</text>
<line class="fl" x1="80" y1="84" x2="110" y2="84" marker-end="url(#aa3)"/><line class="fl" x1="194" y1="84" x2="320" y2="84" marker-end="url(#aa3)"/><line class="fl" x1="400" y1="84" x2="520" y2="84" marker-end="url(#aa3)"/><line class="fl" x1="604" y1="84" x2="636" y2="84" marker-end="url(#aa3)"/>
<text x="357" y="44" text-anchor="middle" class="mono">VXLAN VNI 10, 10.0.1.3 to 10.0.1.1</text>
<text x="152" y="128" text-anchor="middle" class="lbl">encap</text><text x="360" y="128" text-anchor="middle" class="lbl">ECMP route</text><text x="562" y="128" text-anchor="middle" class="lbl">decap</text>
<text x="350" y="160" text-anchor="middle" class="lbl">Known unicast, straight to the right VTEP. No flood, no guesswork.</text>
<defs><marker id="aa3" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Step 3. The data packet is known unicast from the first frame, because the control plane already placed VM A. Compare this with the flood-first behavior of Chapter 6.</figcaption>
</figure>

Put the three steps side by side with Chapter 6 and the improvement is stark. There was no flooding to discover VM A, no flooding to resolve its ARP, and no first-packet penalty. Everything the data plane needed, the control plane had already delivered.

## Configuring an EVPN-VXLAN leaf

Here is what the leaf configuration looks like, first in vendor-neutral terms, then in FRR. The pieces are: the BGP EVPN address family, a VXLAN interface mapping VNI 10 to a bridge, and the setting that advertises locally learned hosts into EVPN.

Vendor-neutral outline on Leaf 1:

```
# 1. map the tenant VLAN/bridge to VNI 10 on a VXLAN interface
vxlan vni 10 -> local VTEP source = loopback 10.0.1.1

# 2. BGP: add the EVPN address family on the existing sessions
router bgp 65101
  neighbor <spine> remote-as 65000
  address-family l2vpn evpn
    neighbor <spine> activate
    advertise-all-vni          # advertise locally learned MAC/IP into EVPN
```

The same in **FRR**, which uses the Linux bridge and a VXLAN netdevice underneath:

```
router bgp 65101
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.2 remote-as 65000
 !
 address-family l2vpn evpn
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.2 activate
  advertise-all-vni
 exit-address-family
```

In FRR the VNI-to-bridge mapping lives in the Linux networking config rather than in `vtysh`. A typical setup creates a VXLAN device bound to the loopback and enslaves it to a bridge:

```
ip link add vxlan10 type vxlan id 10 dstport 4789 local 10.0.1.1 nolearning
ip link set vxlan10 master br10
```

The `nolearning` flag is the quiet star of that line. It turns off data-plane MAC learning on the VXLAN device, because EVPN, not flooding, is now the source of truth for where MACs live. On **SONiC**, the same intent is expressed through its configuration database and CLI, which create the VXLAN tunnel, map VLANs to VNIs, and enable the `l2vpn evpn` family on BGP, but the model is identical: a VXLAN data plane driven by an EVPN control plane.

## Still only bridging

Everything so far keeps traffic inside **one** segment. VM A and VM B were in the same subnet and the same VNI, so the fabric only had to **bridge** between them. Real data centers have many subnets, and traffic constantly needs to move **between** them, which is routing, not bridging. How EVPN-VXLAN routes between subnets, using the L3VNI and type 5 routes hinted at earlier, is the subject of the next chapter.
