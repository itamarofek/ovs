l2-relay Xconnect
=================
This document describes the hardware_vtep l2-relay Xconnect feature.
The aim of this feature is to support the usage of hardware vtep switch as
an inter-cloud gateway, which is used to connect two separated l2 broadcast
domains. This document will also explain the logic behind the addition of the 
new per-tunnel tunnel-key in the hardware_vtep schema.  

The introduction of the relay tunnel, does not change the current logic of 
hardware_vtep, it does however introduce new logic related to iner-cloud
connectivity.

ovs-vtep
---------
The ovs-vtep is a Python script that monitors the ovsdb hardware_vtep schema 
and plants flows in the ovs according to the information in the schema.

unknwon-dst
-----------
A directive for handling L2 "BUM", broadcast, multicast and unknown unicast,
traffic.  "BUM" traffic is sent to all VTEP ports of a logical switch
referenced by an "unknown-dst" address. There are different modes to 
replicate these packets, see the "Multicast and Unknown unicast traffic"
paragraph below.


Network layout
==============

    virtual network 1                 shared  network         virtual network 2 
    +------------+                                                  +------------+
    |Compute Host|   VNI=1                                    VNI=2 |Compute Host|
    |   +--+     <---------+                                +------->   +--+     |
    |   |vm|     |         |                                |       |   |vm|     |
    |   +--+     |         |           L3 network           |       |   +--+     |
    +---^--------+         |                                |       +---------^--+
        |          +-------v--------+      X     +----------v-----+           |   
        |      +---> hardware_vtep  |      X     | hardware_vtep  |           |   
        | VNI=1|   | logical switch |      X     | logical switch <-----+     |   
        |      |   | (tunnel_key 1) |==<<==X=>>==| (tunnel_key 2) |     |VNI=2|   
        |      |   |   +-+     +-+  |      X     |   +-+      +-+ |     |     |   
        |      |   |   |-|     |-|  |      X     |   |-|      |-| |     |     |   
    +---v------v-+ +----------------+      X     +----------------+     |     |   
    |Compute Host| vlan2|       |vlan5           vlan9|        |vlan21  |     |   
    |   +--+     |      |       |    relay vxlan      |        |    +---v-----v--+
    |   |vm|     |      |       |      VNI=100        |        |    |Compute Host|
    |   +--+     |      |       |                     |        |    |   +--+     |
    +------------+    +-v-+   +-v-+                 +-v-+    +-v-+  |   |vm|     |
                      |   |   |   |                 |   |    |   |  |   +--+     |
                      |   |   |   |                 |   |    |   |  +------------+
                      +---+   +---+                 +---+    +---+                
                    Bare metal elements             Bare metal elements

Logical switch
===============
In a cloud architecture, there is sometimes need to connect virtual machines
and physical machines to the same L2 broadcast domain.
A logical switch is an entity representing an l2 virtual overlay network,
identified by a shared tunnel key. This tunnel key (VxLAN VNI) is shared among
all overlay virtual tunnel endpoints (VTEP) of the switch.
The logical switch binds physical ports with either identical or different
"VLAN" tags to the "VxLAN overlay" network.

In a multi-cloud architecture, it may be useful to establish a cross-cloud 
l2 broadcast domain. The extended hardware vtep uses a relay l2 tunnel,
which is a tunnel with an explicit tunnel-key property. The tunnel-key propery 
is used to map each overlay network (logical switch tunnel-key) in each cloud tothe tunnel-key of the relay tunnel.

The mapping to a remote logical switch is done by defining the same tunnel key
in both ends of the the relay tunnel. This tunnel key (VxLAN VNI) is a
property of the relay tunnel itself.

To support the above tunnel behevior, a new kind of VTEP port is logic is
introduced, defining two VTEP port groups. One group represents the existing
VTEP ports of the local l2 overlay network, and another new group represents theindividual l2 relay VTEPS.

Multicast and Unknown unicast traffic
=====================================
Currently "BUM" traffic to the overlay networks is handled by two replication
modes:

  - "source_node" mode: The packets originated from physical port
    are replicated on all the VTEPs ports pointed by unknown-dst, and flooded
    to all the physical bound ports.

  - "service_node" mode: The packets originated from physical port are
    forwarded to only a selected single service node from the unknown-dst ports
    (the service node responsible for "BUM" propagation to the overlay network),
    and flooded to all the physical bound ports.

In either of the replication modes BUM traffic originated from a VTEP port is
flooded only to all physical ports.

Considering the new l2-relay VTEPs ports group, relay related BUM traffic is 
handled as follows:
  - BUM packets from Physical Port and from local overlay network are replicated
    to the unknown-dst l2-relay ports. 
       The reason is that a relay VTEP is not a part of the local managed
       network and BUM traffic to the relay VTEPs cannot be suppressed by a
       local service node.
  - BUM packets from relay VTEPs are handled according to the replicating mode
    and not forwarded/replicated to the l2-relay ports.
       It is assumed that relay tunnels do not interconnect remote networks.

To sum it up, the role of the l2-relay is merely to connect two distinct 
broadcast domains, it will otherwise never be used as a switching fabric among
the different remote l2 networks.

ovs-vtep manifested changes
===========================

1. The code need to distinguish between overlay (mesh) VTEPs, and relay VTEPs.
2. BUM traffic
    * BUM traffic from either mesh or bare metal network is replicated on all
      relay 'unknown-dst' VTEP.
    * BUM traffic from relay VTEP, is handled according to the replicating mode
      and is not forwarded/replicated to the l2-relay ports.


