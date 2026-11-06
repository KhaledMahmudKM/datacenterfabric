# Appendix A: Reference Tables

A single place for the facts and formats you will look up most often. Everything here is explained in the chapters; this is the quick-reference version.

## VXLAN at a glance

| Property | Value |
| --- | --- |
| Defined in | RFC 7348 |
| Encapsulation | MAC-in-UDP (a full Ethernet frame inside UDP) |
| Outer transport | UDP |
| Well-known UDP destination port | 4789 |
| Outer UDP source port | hash of the inner flow (for ECMP spreading) |
| VXLAN header size | 8 bytes |
| Segment identifier | VNI, 24 bits |
| Number of segments | about 16 million |
| Typical added overhead | about 50 bytes (plan MTU accordingly) |
| Tunnel endpoint | VTEP, addressed by its underlay loopback |

## The VXLAN packet, outside to inside

| Order | Header | Key contents |
| --- | --- | --- |
| 1 | Outer Ethernet | next-hop MACs across the fabric |
| 2 | Outer IP | source and destination VTEP loopbacks |
| 3 | Outer UDP | destination port 4789, source port = flow hash |
| 4 | VXLAN | flags and the 24-bit VNI |
| 5 | Inner Ethernet frame | the original tenant frame and payload |

## EVPN at a glance

| Property | Value |
| --- | --- |
| Base specification | RFC 7432 |
| VXLAN overlay specification | RFC 8365 |
| Underlay routing reference | RFC 7938 (BGP in data centers) |
| Control-plane protocol | MP-BGP |
| Address family (AFI) | 25 (L2VPN) |
| Subsequent address family (SAFI) | 70 (EVPN) |
| VXLAN encapsulation extended community | type 8 |
| Route Distinguisher (RD) | 64 bits, kept unique per VTEP per segment |
| Route Target (RT) | extended community controlling import and export |
| Ethernet Segment Identifier (ESI) | 10 bytes |

## The five EVPN route types

| Type | Name | Purpose |
| --- | --- | --- |
| 1 | Ethernet Auto-Discovery (A-D) | multihoming: aliasing and fast mass-withdrawal failover |
| 2 | MAC/IP Advertisement | advertise a host MAC and optionally its IP (host learning, ARP suppression) |
| 3 | Inclusive Multicast Ethernet Tag (IMET) | VTEP discovery per VNI, builds the BUM flood list |
| 4 | Ethernet Segment | segment peer discovery and designated-forwarder election |
| 5 | IP Prefix | advertise a subnet or external prefix for routing |

## Bridging and routing VNIs

| Term | Meaning |
| --- | --- |
| L2VNI | a bridging segment, one per subnet |
| L3VNI | a routing (transit) segment, one per tenant VRF |
| VRF | a tenant's private routing table |
| IRB | integrated routing and bridging on a leaf |
| Symmetric IRB | route in to the L3VNI, cross the fabric, route out; leaves need only their own subnets plus the L3VNI (the common choice) |
| Asymmetric IRB | ingress leaf routes then bridges into the destination L2VNI; every leaf needs every subnet's L2VNI (does not scale as well) |
| Distributed anycast gateway | same gateway IP and MAC on every leaf, so routing happens at the ingress leaf |

## Underlay BGP numbering (example scheme)

| Element | Example |
| --- | --- |
| Spine ASN | 65000 (shared) |
| Leaf ASNs | 65101, 65102, 65103, ... (unique per leaf) |
| Private 16-bit ASN range | 64512 to 65534 |
| Loopbacks | 10.0.0.0/24 spines, 10.0.1.0/24 leaves (each a /32) |
| Point-to-point links | /31 subnets, or BGP unnumbered over IPv6 link-local |
| Loop prevention | eBGP AS-path rule plus unique leaf ASNs |
| ECMP | `maximum-paths` set to at least the number of spines |

## Handy FRR verification commands

| Goal | Command |
| --- | --- |
| Interfaces and MTU | `show interface brief` |
| Reach a remote loopback over ECMP | `show ip route 10.0.1.3/32` |
| VTEP-to-VTEP reachability | `ping 10.0.1.3 source 10.0.1.1` |
| EVPN BGP sessions | `show bgp l2vpn evpn summary` |
| All EVPN routes | `show bgp l2vpn evpn` |
| VNIs and their state | `show evpn vni` |
| Details and VTEP list for one VNI | `show evpn vni 10` |
| MACs learned in a VNI | `show evpn mac vni 10` |

## Minimal FRR leaf configuration (reference)

Underlay plus EVPN on one leaf, condensed from Chapters 3, 9, and 10:

```
router bgp 65101
 bgp router-id 10.0.1.1
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.2 remote-as 65000
 !
 address-family ipv4 unicast
  network 10.0.1.1/32
  maximum-paths 64
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.2 activate
  advertise-all-vni
 exit-address-family
!
vrf TENANT_A
 vni 10999
exit-vrf
```

The VXLAN device and bridge mapping, in Linux networking terms:

```
ip link add vxlan10 type vxlan id 10 dstport 4789 local 10.0.1.1 nolearning
ip link set vxlan10 master br10
```
