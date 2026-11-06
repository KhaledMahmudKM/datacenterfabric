# Chapter 13: Operating and Troubleshooting

A built fabric still has to be run. This chapter covers the habits and commands for operating EVPN-VXLAN, and a layered method for finding faults when something breaks. The single most useful idea is to always troubleshoot from the **bottom up**: confirm the underlay before you suspect the overlay.

## The layered mental model

Everything in this book stacks. When traffic fails, the fault lives in one of these layers, and they depend on each other from the bottom.

<figure class="diagram">
<svg viewBox="0 0 640 300" style="width:100%;height:auto;max-width:640px" role="img" aria-label="A five-layer stack: physical links at the bottom, then the IP underlay, then BGP EVPN sessions, then EVPN routes, then the VXLAN data plane at the top, each depending on the one below." xmlns="http://www.w3.org/2000/svg">
<style>
.p1{ font-family:var(--sans),sans-serif; }
.p1 .l0{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.p1 .l1{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.4; }
.p1 .l2{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.4; }
.p1 .l3{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.4; }
.p1 .l4{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.p1 .t{ fill:var(--text); font-size:13px; font-weight:700; }
.p1 .s{ fill:var(--text-dim); font-size:13px; }
.p1 .arr{ stroke:var(--accent); stroke-width:1.6; fill:none; }
.p1 .lbl{ fill:var(--text-faint); font-size:12.5px; }
</style>
<g class="p1">
<rect x="120" y="30" width="400" height="40" rx="6" class="l4"/><text x="150" y="48" class="t">5. VXLAN data plane</text><text x="150" y="63" class="s">frames encapsulated and delivered</text>
<rect x="120" y="82" width="400" height="40" rx="6" class="l3"/><text x="150" y="100" class="t">4. EVPN routes</text><text x="150" y="115" class="s">type 2 / 3 / 5 present and correct</text>
<rect x="120" y="134" width="400" height="40" rx="6" class="l2"/><text x="150" y="152" class="t">3. BGP EVPN sessions</text><text x="150" y="167" class="s">l2vpn evpn neighbors established</text>
<rect x="120" y="186" width="400" height="40" rx="6" class="l1"/><text x="150" y="204" class="t">2. IP underlay</text><text x="150" y="219" class="s">loopbacks reachable over ECMP</text>
<rect x="120" y="238" width="400" height="40" rx="6" class="l0"/><text x="150" y="256" class="t">1. Physical links</text><text x="150" y="271" class="s">cables up, interfaces up, MTU set</text>
<path class="arr" d="M70 258 L70 50" marker-end="url(#ap1)"/><text x="40" y="160" class="lbl" transform="rotate(-90 40 160)">check in this order</text>
<defs><marker id="ap1" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9 z" fill="var(--accent)"/></marker></defs>
</g>
</svg>
<figcaption>The troubleshooting stack. A broken overlay is very often a broken underlay. Always verify from the bottom up.</figcaption>
</figure>

## Layer by layer, with commands

The commands below are **FRR** form, which runs on many open fabrics and in a container lab. Other platforms use different words for the same checks, so focus on what each command confirms.

**Layer 1, physical and MTU.** Confirm interfaces are up, and confirm the **MTU** is raised. This is the classic silent killer: VXLAN adds about 50 bytes, so if the fabric MTU is not increased, full-size tenant frames are dropped after encapsulation while small packets like pings pass. A fabric that "works for ping but not for real traffic" is almost always an MTU problem.

```
show interface brief
```

**Layer 2, the underlay.** Confirm each switch can reach remote **loopbacks**, and over **multiple paths**. If a VTEP's loopback is not reachable, no tunnel to it can form.

```
show ip route 10.0.1.3/32        # should show multiple ECMP next hops
ping 10.0.1.3 source 10.0.1.1    # VTEP to VTEP loopback reachability
```

**Layer 3, BGP sessions.** Confirm the **EVPN address family** sessions are established, separately from the IPv4 ones. A common mistake is a session that is up for IPv4 unicast but was never activated for `l2vpn evpn`.

```
show bgp l2vpn evpn summary      # neighbors should be Established
```

**Layer 4, EVPN routes.** Confirm the expected routes exist. Is the missing host's **type 2** route present? Are the VTEPs' **type 3** routes all there for the VNI? Is the subnet's **type 5** route present for inter-subnet problems?

```
show bgp l2vpn evpn              # all received EVPN routes
show evpn mac vni 10             # MACs learned in VNI 10
show evpn vni                    # VNIs and their type/VTEP state
```

**Layer 5, the VXLAN data plane.** Confirm the VNI is up, mapped to the right bridge, and sourced from the right loopback, and check the remote VTEP flood list.

```
show evpn vni 10                 # VNI 10 details, VTEP list
```

<figure class="diagram">
<svg viewBox="0 0 700 250" style="width:100%;height:auto;max-width:700px" role="img" aria-label="A troubleshooting decision flow: start at loopback reachability, then BGP EVPN session, then the type 2 route, then MTU, ending at the likely fault at each failing step." xmlns="http://www.w3.org/2000/svg">
<style>
.p2{ font-family:var(--sans),sans-serif; }
.p2 .q{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.5; }
.p2 .fix{ fill:var(--panel); stroke:var(--border); stroke-width:1.4; }
.p2 .yes{ stroke:var(--accent); stroke-width:1.8; fill:none; }
.p2 .no{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.p2 .s{ fill:var(--text-dim); font-size:13px; }
.p2 .lbl{ fill:var(--accent); font-size:12.5px; font-weight:700; }
.p2 .nol{ fill:var(--text-faint); font-size:13px; font-weight:700; }
.p2 .fx{ fill:var(--text-dim); font-size:13px; }
</style>
<g class="p2">
<rect x="20" y="30" width="130" height="44" rx="8" class="q"/><text x="85" y="50" text-anchor="middle" class="s">loopback</text><text x="85" y="64" text-anchor="middle" class="s">reachable?</text>
<rect x="190" y="30" width="130" height="44" rx="8" class="q"/><text x="255" y="50" text-anchor="middle" class="s">EVPN session</text><text x="255" y="64" text-anchor="middle" class="s">Established?</text>
<rect x="360" y="30" width="130" height="44" rx="8" class="q"/><text x="425" y="50" text-anchor="middle" class="s">type 2 route</text><text x="425" y="64" text-anchor="middle" class="s">present?</text>
<rect x="530" y="30" width="140" height="44" rx="8" class="q"/><text x="600" y="50" text-anchor="middle" class="s">MTU raised</text><text x="600" y="64" text-anchor="middle" class="s">end to end?</text>
<line class="yes" x1="150" y1="52" x2="190" y2="52" marker-end="url(#ap2)"/><text x="170" y="44" text-anchor="middle" class="lbl">yes</text>
<line class="yes" x1="320" y1="52" x2="360" y2="52" marker-end="url(#ap2)"/><text x="340" y="44" text-anchor="middle" class="lbl">yes</text>
<line class="yes" x1="490" y1="52" x2="530" y2="52" marker-end="url(#ap2)"/><text x="510" y="44" text-anchor="middle" class="lbl">yes</text>
<rect x="20" y="130" width="130" height="62" rx="6" class="fix"/><text x="85" y="150" text-anchor="middle" class="fx">fix underlay:</text><text x="85" y="167" text-anchor="middle" class="fx">link, BGP ipv4,</text><text x="85" y="184" text-anchor="middle" class="fx">route-map</text>
<rect x="190" y="130" width="130" height="62" rx="6" class="fix"/><text x="255" y="150" text-anchor="middle" class="fx">activate</text><text x="255" y="167" text-anchor="middle" class="fx">l2vpn evpn on</text><text x="255" y="184" text-anchor="middle" class="fx">the neighbor</text>
<rect x="360" y="130" width="130" height="62" rx="6" class="fix"/><text x="425" y="150" text-anchor="middle" class="fx">check RT import,</text><text x="425" y="167" text-anchor="middle" class="fx">advertise-all-vni,</text><text x="425" y="184" text-anchor="middle" class="fx">VNI mapping</text>
<rect x="530" y="130" width="140" height="62" rx="6" class="fix"/><text x="600" y="150" text-anchor="middle" class="fx">raise MTU on</text><text x="600" y="167" text-anchor="middle" class="fx">every fabric</text><text x="600" y="184" text-anchor="middle" class="fx">link and uplink</text>
<line class="no" x1="85" y1="74" x2="85" y2="130" marker-end="url(#ap2b)"/><text x="105" y="104" class="nol">no</text>
<line class="no" x1="255" y1="74" x2="255" y2="130" marker-end="url(#ap2b)"/><text x="275" y="104" class="nol">no</text>
<line class="no" x1="425" y1="74" x2="425" y2="130" marker-end="url(#ap2b)"/><text x="445" y="104" class="nol">no</text>
<line class="no" x1="600" y1="74" x2="600" y2="130" marker-end="url(#ap2b)"/><text x="620" y="104" class="nol">no</text>
<text x="350" y="214" text-anchor="middle" class="s">Walk left to right. The first "no" points straight at the layer to fix.</text>
<defs><marker id="ap2" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--accent)"/></marker><marker id="ap2b" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="var(--text-faint)"/></marker></defs>
</g>
</svg>
<figcaption>A troubleshooting flow for "host A cannot reach host B." The first failing check names the layer at fault, which saves you from guessing.</figcaption>
</figure>

## Common faults and what they look like

A few patterns recur often enough to recognize on sight.

- **Ping works, real traffic fails.** MTU. Encapsulated full-size frames exceed the fabric MTU and are dropped. Raise the MTU everywhere.
- **One VNI works, another does not.** Route target or VNI mapping. The broken VNI is probably missing from `advertise-all-vni`, or its RT import does not match, so its type 2 and type 3 routes never arrive.
- **A host is reachable from some leaves but not others.** Likely a missing or stale type 2 route on the affected leaves, or an RT mismatch on just those leaves.
- **Duplicate packets or a flapping MAC on a multihomed server.** A multihoming misconfiguration: mismatched ESI between the two leaves, or a broken DF election. Check the type 4 and type 1 state for that segment.
- **Inter-subnet fails while same-subnet works.** A routing problem, not a bridging one. Check the L3VNI, the VRF, and the type 5 routes, since bridging within the L2VNI is clearly fine.

## Good operational habits

Beyond fault-finding, a few practices keep a fabric healthy over time.

- **Monitor the control plane, not just links.** Track BGP session state and EVPN route counts. A sudden drop in received type 2 routes is an early warning that something upstream broke.
- **Watch for MAC moves.** A MAC that rapidly reappears behind different VTEPs, a **MAC flap**, usually means a loop, a duplicate address, or a misbehaving dual-homed server. Most platforms can alarm on this.
- **Keep configuration in version control and generated.** From Chapter 12, configs come from templates. Changes go through the template and the data files, never by hand on a live switch, so the fabric never drifts from its source of truth.
- **Test in a lab first.** Because the whole stack is standard BGP and Linux networking, you can build the exact topology in ContainerLab with FRR or SONiC images and validate a change before it touches production. Everything in this book is reproducible that way.

## Closing the loop

That is the whole arc of the book. A routed spine-leaf fabric gives predictable, scale-out bandwidth. VXLAN tunnels tenant Layer 2 across it. EVPN provides a BGP control plane so the tunnels are driven by knowledge instead of flooding. Anycast gateways and type 5 routes add routing at the edge, and types 1 and 4 make multihoming safe. Operating it all comes back to one disciplined idea: understand the layers, and check them from the bottom up.

The Appendix gathers the reference tables you will reach for most, and the Glossary defines every term in one place.
