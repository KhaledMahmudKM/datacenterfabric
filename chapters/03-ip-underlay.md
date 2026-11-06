# Chapter 3: The IP Underlay

The fabric from Chapter 2 is a set of cables. The **underlay** is the routing that brings it to life. Its one job is humble but essential: make sure every switch can reach every other switch's address over all available paths. Nothing about tenants, VLANs, or virtual machines lives here yet. The underlay just moves packets between switches.

## Route the links, do not bridge them

The first and most important decision is that every link between a leaf and a spine is a **routed** (Layer 3) link, not a switched (Layer 2) one. Each end gets an IP address, and the switches run a routing protocol over it.

This is what finally kills spanning tree. There is no Layer 2 domain stretched across the fabric, so there are no Layer 2 loops to prevent, so STP has nothing to block. Loop prevention becomes the job of the routing protocol, which is built to use many paths at once rather than prune them away.

<figure class="diagram">
<svg viewBox="0 0 680 250" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Each leaf-spine link is a routed point-to-point subnet, and each switch has a loopback address." xmlns="http://www.w3.org/2000/svg">
<style>
.u1{ font-family:var(--sans),sans-serif; }
.u1 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.u1 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.u1 .ln{ stroke:var(--border); stroke-width:1.4; fill:none; }
.u1 .s{ fill:var(--text-dim); font-size:13px; }
.u1 .ip{ fill:var(--text-faint); font-size:12.5px; font-family:var(--mono),monospace; }
.u1 .lo{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.u1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="u1">
<line class="ln" x1="220" y1="60" x2="140" y2="150"/><line class="ln" x1="220" y1="60" x2="540" y2="150"/><line class="ln" x1="460" y1="60" x2="140" y2="150"/><line class="ln" x1="460" y1="60" x2="540" y2="150"/>
<rect x="170" y="36" width="100" height="26" rx="6" class="spine"/><text x="220" y="54" text-anchor="middle" class="s">Spine 1</text>
<text x="220" y="30" text-anchor="middle" class="lo">lo 10.0.0.1</text>
<rect x="410" y="36" width="100" height="26" rx="6" class="spine"/><text x="460" y="54" text-anchor="middle" class="s">Spine 2</text>
<text x="460" y="30" text-anchor="middle" class="lo">lo 10.0.0.2</text>
<rect x="90" y="150" width="100" height="26" rx="6" class="leaf"/><text x="140" y="168" text-anchor="middle" class="s">Leaf 1</text>
<text x="140" y="196" text-anchor="middle" class="lo">lo 10.0.1.1</text>
<rect x="490" y="150" width="100" height="26" rx="6" class="leaf"/><text x="540" y="168" text-anchor="middle" class="s">Leaf 2</text>
<text x="540" y="196" text-anchor="middle" class="lo">lo 10.0.1.2</text>
<text x="174" y="98" text-anchor="end" class="ip">10.1.0.0/31</text><text x="506" y="98" class="ip">10.1.0.6/31</text>
<text x="350" y="230" text-anchor="middle" class="lbl">Every link is its own tiny /31 subnet. Every switch has a /32 loopback that identifies it.</text>
</g>
</svg>
<figcaption>The routed underlay. Point-to-point /31 links carry the routing protocol, and loopbacks are the stable addresses switches use to reach each other.</figcaption>
</figure>

Two kinds of address appear here. The **point-to-point links** use tiny subnets, usually a `/31`, which holds exactly two usable addresses, one per end, and wastes nothing. The **loopback** is a `/32` address that belongs to the switch itself rather than to any one link. Because a loopback stays up as long as the switch is alive, and is reachable over every uplink, it is the address everything else uses to find a switch. In later chapters the VXLAN tunnel endpoints will all be loopbacks.

## Why BGP runs the underlay

Any routing protocol could fill a fabric with routes, and OSPF or IS-IS are used in some designs. But the dominant choice, documented in **RFC 7938**, is the **Border Gateway Protocol (BGP)**, specifically **eBGP** (external BGP) between switches. Students usually meet BGP as the protocol of the global internet, so running it inside a single data center can feel surprising. The reasons are practical:

- **It scales.** BGP was built to carry the internet's routing table, so a data center's routes are trivial for it.
- **It is simple to reason about.** Each switch is its own routing domain, and routes propagate hop by hop with clear, predictable policy.
- **It carries everything on one protocol.** As we will see, the same BGP sessions that build the underlay will later carry the EVPN overlay information too. One protocol, one set of sessions.

## One AS number per switch

In eBGP, every router belongs to an **autonomous system** identified by an **AS number (ASN)**. eBGP speakers in different AS numbers exchange routes. The clean design for a fabric gives **each leaf its own unique ASN** and lets the **spines share a single ASN** (or each take its own, depending on the scheme). Private ASNs in the 16-bit range 64512 to 65534, or the larger 32-bit private range, keep these numbers out of the public internet's way.

<figure class="diagram">
<svg viewBox="0 0 680 250" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Each spine and each leaf has its own autonomous system number, and eBGP sessions run on every link." xmlns="http://www.w3.org/2000/svg">
<style>
.u2{ font-family:var(--sans),sans-serif; }
.u2 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.u2 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.u2 .ln{ stroke:var(--accent); stroke-width:1.8; fill:none; }
.u2 .s{ fill:var(--text-dim); font-size:13px; }
.u2 .asn{ fill:var(--accent); font-size:13px; font-weight:700; }
.u2 .lbl{ fill:var(--text-faint); font-size:12.5px; }
.u2 .tag{ fill:var(--text-faint); font-size:13px; }
</style>
<g class="u2">
<line class="ln" x1="200" y1="62" x2="110" y2="150"/><line class="ln" x1="200" y1="62" x2="290" y2="150"/><line class="ln" x1="200" y1="62" x2="470" y2="150"/>
<line class="ln" x1="440" y1="62" x2="110" y2="150"/><line class="ln" x1="440" y1="62" x2="290" y2="150"/><line class="ln" x1="440" y1="62" x2="470" y2="150"/>
<rect x="150" y="38" width="100" height="26" rx="6" class="spine"/><text x="200" y="56" text-anchor="middle" class="s">Spine 1</text><text x="200" y="30" text-anchor="middle" class="asn">AS 65000</text>
<rect x="390" y="38" width="100" height="26" rx="6" class="spine"/><text x="440" y="56" text-anchor="middle" class="s">Spine 2</text><text x="440" y="30" text-anchor="middle" class="asn">AS 65000</text>
<rect x="66" y="150" width="88" height="26" rx="6" class="leaf"/><text x="110" y="168" text-anchor="middle" class="s">Leaf 1</text><text x="110" y="196" text-anchor="middle" class="asn">AS 65101</text>
<rect x="246" y="150" width="88" height="26" rx="6" class="leaf"/><text x="290" y="168" text-anchor="middle" class="s">Leaf 2</text><text x="290" y="196" text-anchor="middle" class="asn">AS 65102</text>
<rect x="426" y="150" width="88" height="26" rx="6" class="leaf"/><text x="470" y="168" text-anchor="middle" class="s">Leaf 3</text><text x="470" y="196" text-anchor="middle" class="asn">AS 65103</text>
<text x="536" y="100" class="tag">eBGP on</text><text x="536" y="117" class="tag">every link</text>
<text x="340" y="232" text-anchor="middle" class="lbl">Spines share one ASN; each leaf has a unique one. This shapes how routes flow and prevents loops.</text>
</g>
</svg>
<figcaption>eBGP numbering for a fabric. Unique leaf ASNs plus a shared spine ASN give loop-free, predictable propagation.</figcaption>
</figure>

This numbering is not just bookkeeping. BGP rejects any route whose AS-path already contains its own ASN, which is the loop-prevention rule. Giving each leaf a unique ASN and the spines a shared one means a route learned from one leaf is advertised by the spines to the other leaves, but never bounced back to the leaf it came from, and never looped between spines. The topology and the numbering together make the fabric loop-free without any spanning tree.

## What the underlay advertises

In the underlay, switches advertise just enough to reach each other. At minimum, every switch advertises its **loopback /32** into BGP. The spines relay these, so every leaf learns a route to every other leaf's loopback, over every spine, as equal-cost paths. That is the ECMP from Chapter 2, now expressed as real routes.

<figure class="diagram">
<svg viewBox="0 0 680 210" style="width:100%;height:auto;max-width:680px" role="img" aria-label="Leaf 1 advertises its loopback, spines relay it, and Leaf 3 installs two equal-cost routes to reach it." xmlns="http://www.w3.org/2000/svg">
<style>
.u3{ font-family:var(--sans),sans-serif; }
.u3 .spine{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.u3 .leaf{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.u3 .fl{ stroke:var(--accent); stroke-width:2.2; fill:none; }
.u3 .s{ fill:var(--text-dim); font-size:13px; }
.u3 .mono{ fill:var(--text-faint); font-size:12.5px; font-family:var(--mono),monospace; }
.u3 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="u3">
<line class="fl" x1="110" y1="92" x2="260" y2="46" marker-end="url(#au3)"/><line class="fl" x1="110" y1="92" x2="260" y2="92" marker-end="url(#au3)"/>
<line class="fl" x1="350" y1="46" x2="500" y2="92" marker-end="url(#au3)"/><line class="fl" x1="350" y1="92" x2="500" y2="92" marker-end="url(#au3)"/>
<rect x="260" y="33" width="90" height="24" rx="6" class="spine"/><text x="305" y="50" text-anchor="middle" class="s">Spine 1</text>
<rect x="260" y="80" width="90" height="24" rx="6" class="spine"/><text x="305" y="97" text-anchor="middle" class="s">Spine 2</text>
<rect x="30" y="80" width="80" height="24" rx="6" class="leaf"/><text x="70" y="97" text-anchor="middle" class="s">Leaf 1</text>
<rect x="500" y="80" width="80" height="24" rx="6" class="leaf"/><text x="540" y="97" text-anchor="middle" class="s">Leaf 3</text>
<text x="70" y="70" text-anchor="middle" class="mono">10.0.1.1/32</text>
<text x="305" y="138" text-anchor="middle" class="lbl">"reach 10.0.1.1 via Spine 1  and  via Spine 2"</text>
<text x="540" y="138" text-anchor="middle" class="mono">2 equal paths</text>
<text x="305" y="182" text-anchor="middle" class="lbl">Leaf 3 installs both next hops and load-balances across them with ECMP.</text>
<defs><marker id="au3" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>Loopback reachability through BGP. Each leaf ends up with one equal-cost route per spine to every other leaf.</figcaption>
</figure>

## A minimal underlay configuration

Here is the idea in vendor-neutral terms, then in FRR, which you can run in a container lab. The leaf advertises its loopback and peers with both spines.

Vendor-neutral, on Leaf 1:

```
router bgp 65101
  router-id 10.0.1.1
  neighbor <spine-1-link-ip> remote-as 65000
  neighbor <spine-2-link-ip> remote-as 65000
  address-family ipv4 unicast
    network 10.0.1.1/32        # advertise my loopback
    maximum-paths 64           # install all equal-cost paths (ECMP)
```

The same configuration in **FRR** (the `frr.conf` or `vtysh` form):

```
router bgp 65101
 bgp router-id 10.0.1.1
 neighbor 10.1.0.0 remote-as 65000
 neighbor 10.1.0.2 remote-as 65000
 !
 address-family ipv4 unicast
  network 10.0.1.1/32
  maximum-paths 64
 exit-address-family
```

Two details matter for a fabric. The `maximum-paths` setting must be high enough to install one path per spine, or ECMP collapses to a single link. And many operators enable **BGP unnumbered**, an FRR and SONiC feature that forms eBGP sessions over IPv6 link-local addresses so you never assign or track the point-to-point `/31` addresses by hand. That removes most of the per-link addressing above and makes the fabric far easier to automate.

## The underlay is deliberately dumb

It is worth stating plainly what the underlay does not do. It does not know about VLANs, tenants, MAC addresses, or virtual machines. It carries no customer traffic in its own right. It simply provides fast, redundant, loop-free reachability between switch loopbacks.

That narrowness is a feature. The underlay is stable and rarely changes, because adding or moving a server does not touch it. All the churn of a live data center, the virtual machines coming and going and moving between hosts, is pushed up into a separate layer that rides on top of this one. That upper layer is the **overlay**, and building it is where the rest of the book goes. The next chapter explains why we need it at all.
