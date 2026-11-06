# Glossary

**Aliasing.** An EVPN feature, carried by the type 1 route, that tells remote VTEPs a multihomed host is reachable through all of its leaves, so they can load-balance across them.

**Anycast gateway (distributed).** The practice of giving every leaf the same gateway IP and MAC for a subnet, so a host always routes at its own local leaf and a moving VM keeps its gateway.

**ARP suppression.** Answering a local ARP request at the leaf, from EVPN-learned IP-to-MAC bindings, so the broadcast never floods the fabric.

**AS, ASN (Autonomous System, AS Number).** A routing domain and its numeric identifier in BGP. Fabrics give each leaf a unique ASN and the spines a shared one.

**Asymmetric IRB.** An IRB model where the ingress leaf routes and then bridges into the destination subnet's L2VNI, requiring every leaf to hold every subnet. Contrast with symmetric IRB.

**BGP (Border Gateway Protocol).** The routing protocol used for both the underlay and, as EVPN, the overlay control plane. eBGP is BGP between different AS numbers.

**Border leaf.** A leaf dedicated to connecting the fabric to external networks, advertising external prefixes in as type 5 routes and fabric subnets out.

**BUM (Broadcast, Unknown-unicast, Multicast).** The classes of traffic that must be delivered to every VTEP in a segment, since they have no single known destination.

**CLOS fabric.** A multi-stage non-blocking switching topology, from Charles Clos, that builds large capacity from many small switches. Spine-leaf is a two-stage folded CLOS.

**Decapsulation (decap).** Removing the outer VXLAN headers at the egress VTEP to recover the original frame.

**Designated Forwarder (DF).** The one leaf on a multihomed segment allowed to deliver BUM traffic to the host at a given time, elected using type 4 routes.

**eBGP.** External BGP, run between different AS numbers. The underlay and EVPN both use it in a fabric.

**ECMP (Equal-Cost Multi-Path).** Spreading traffic across several equal-cost paths at once, which is how a fabric uses all its spines.

**Encapsulation (encap).** Wrapping a tenant frame in outer headers (for VXLAN: Ethernet, IP, UDP, and the VXLAN header) at the ingress VTEP.

**ESI (Ethernet Segment Identifier).** A 10-byte value identifying the set of links that attach one multihomed host to several leaves.

**Ethernet Segment.** The group of links from several leaves to the same multihomed host, named by an ESI.

**EVI (EVPN Instance).** One tenant virtual network's worth of EVPN state.

**EVPN (Ethernet VPN).** The BGP-based control plane, defined in RFC 7432 and extended for VXLAN in RFC 8365, that distributes host and segment reachability so VTEPs do not have to flood and learn.

**Flood-and-learn.** The original VXLAN method of discovering hosts by flooding unknown and broadcast traffic and learning MACs from received frames. It has no control plane and scales poorly.

**Head-end replication (ingress replication).** Delivering BUM traffic by having the source VTEP send a separate unicast copy to each remote VTEP in the VNI.

**IMET (Inclusive Multicast Ethernet Tag).** The type 3 route, by which a VTEP announces its membership in a VNI so others can build a BUM flood list.

**IRB (Integrated Routing and Bridging).** A leaf's ability to both bridge within a subnet and route between subnets.

**L2VNI.** A VNI used for bridging, one per subnet.

**L3VNI.** A VNI used for routed transit between subnets, one per tenant VRF, central to symmetric IRB.

**Leaf.** An edge switch in a spine-leaf fabric. Servers connect to leaves, and leaves act as VTEPs.

**Loopback.** A switch's own /32 address, reachable over every uplink and stable as long as the switch is up. VTEP tunnels are built between loopbacks.

**MAC flap.** A MAC address rapidly appearing behind different VTEPs, usually a sign of a loop, a duplicate address, or a multihoming fault.

**Mass withdrawal.** Failing an entire multihomed segment over to the surviving leaf by withdrawing one type 1 route, instead of withdrawing every host's type 2 route.

**MLAG (Multi-Chassis Link Aggregation).** A vendor method of dual-homing a host to two switches that act as one, typically using a dedicated peer link. EVPN multihoming achieves similar ends without that peer link.

**MP-BGP (Multiprotocol BGP).** BGP extended to carry many address families, including L2VPN EVPN.

**MTU (Maximum Transmission Unit).** The largest frame a link will carry. Fabrics raise it because VXLAN adds about 50 bytes, and a too-small MTU silently drops full-size encapsulated frames.

**Multihoming.** Connecting a host to more than one leaf for redundancy.

**Oversubscription.** The ratio of downlink (server) bandwidth to uplink (fabric) bandwidth. Lower ratios mean less contention for east-west traffic.

**Overlay.** The virtual network of tenant segments built on top of the fabric with VXLAN and EVPN.

**Pod.** One spine-leaf fabric within a larger multi-pod design connected by super-spines.

**RD (Route Distinguisher).** A 64-bit value prepended to advertised addresses so routes stay unique across tenants even when their addresses overlap.

**Route reflector (RR).** A BGP speaker, often a spine, that receives routes once and reflects them to all other peers, keeping the number of sessions manageable.

**RT (Route Target).** A BGP extended community that controls which VTEPs import a route, and therefore which tenant it belongs to.

**SONiC.** An open-source network operating system that runs EVPN-VXLAN on merchant-silicon switches.

**Spine.** A middle-tier switch connecting only to leaves. Spines are transit and often act as route reflectors.

**Spine-leaf.** The two-tier fabric topology where every leaf connects to every spine and nothing else.

**Split-horizon filtering.** Dropping a frame that returns to the Ethernet Segment it originated from, preventing a multihomed host from receiving its own flooded traffic.

**Super-spine.** A switch tier connecting several pods in a multi-pod (five-stage) CLOS.

**Symmetric IRB.** The scalable IRB model: route into the L3VNI, cross the fabric in it, route out at egress, so each leaf needs only its own subnets plus the shared L3VNI.

**Three-tier design.** The older access, aggregation, and core topology that spine-leaf replaced.

**Underlay.** The physical routed fabric and its BGP routing, whose only job is to carry outer IP packets between VTEP loopbacks.

**VLAN (Virtual LAN).** A 12-bit Layer 2 segment identifier, limited to 4094 usable values, which VXLAN's VNI supersedes for fabric-wide segmentation.

**VNI (VXLAN Network Identifier).** The 24-bit segment identifier in the VXLAN header, allowing about 16 million segments.

**VRF (Virtual Routing and Forwarding).** A tenant's isolated routing table, tied to an L3VNI for overlay routing.

**VTEP (VXLAN Tunnel Endpoint).** The device, a leaf in a fabric, that encapsulates and decapsulates VXLAN. Its address is its underlay loopback.

**VXLAN (Virtual Extensible LAN).** The MAC-in-UDP encapsulation, defined in RFC 7348, that tunnels Layer 2 frames across a routed fabric using a 24-bit VNI and UDP port 4789.
