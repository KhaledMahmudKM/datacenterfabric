# Data Center Fabric

### Spine-Leaf, VXLAN & EVPN

A concise, diagram-rich guide to how modern data center networks are built. This book explains the leaf-spine fabric, the VXLAN data plane that tunnels tenant traffic across it, and the EVPN control plane that makes the whole thing scale.

It is written for fourth-year network engineering students who already know IP routing, BGP, and Ethernet switching from earlier courses. The goal here is to connect those pieces into the architecture that runs almost every cloud and enterprise data center today.

---

## How this book is organized

**Part I, The Fabric and the Underlay.** Why the old three-tier design ran out of room, how a spine-leaf CLOS fabric replaces it, and how a simple routed underlay built on BGP ties the switches together.

**Part II, VXLAN Overlays.** What an overlay is and why we need one, how the VXLAN packet format tunnels Layer 2 over Layer 3, and the limits of the original flood-and-learn approach.

**Part III, The EVPN Control Plane.** How EVPN replaces flooding with BGP route advertisements, the five route types you will meet in practice, and how routing, multihoming, and redundancy work on top of them.

**Part IV, Design and Operations.** Turning the concepts into a buildable design, then the commands and habits you use to operate and troubleshoot a live fabric.

The **Appendix** collects the reference tables (ports, route types, address formats) in one place, and the **Glossary** defines every acronym.

---

## How to read it

Every chapter leans on diagrams. Read them. A packet walk or a route exchange is far easier to follow as a picture than as a paragraph, and the diagrams are the backbone of each explanation, not decoration.

Configuration snippets are vendor-neutral first, so the idea is clear, and then shown in [FRR](#/14-reference-tables) and SONiC form so you can build the examples yourself in a lab. If you have worked through the ContainerLab material, you can bring up every topology in this book on your own machine.

Let us start with the problem that spine-leaf was invented to solve.
