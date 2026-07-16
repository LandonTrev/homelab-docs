# 2026-07-14 - VLAN Theory Study Session

Study session only, no lab changes. Worked through the Lab Networking Study Guide to reinforce concepts before the DC01 migration into VLAN 10. Format was explain-the-confusing-parts followed by a five question quiz.

## Concepts covered

### The flat network and VLAN 1

- A broadcast does not need a VLAN to exist. Every layer 2 network is a broadcast domain, including the home network.
- The home network is a flat network: one big room with no interior walls. Every device hears every broadcast.
- On a switch there is no such thing as "no VLAN." Untagged traffic belongs to the default VLAN (VLAN 1 on most gear). Out of the factory every port is an untagged member of VLAN 1, which is why the switch behaves like one open room.
- Building VLAN 10 did not create the concept of rooms. It built a second room next to the one that always existed.
- Precision point: a broadcast is stopped by the layer 2 boundary (the VLAN wall), not by subnet numbering. In a well built network segments and subnets line up one to one, but the wall is what physically stops the frame.
### How devices ignore traffic - two different mechanisms

- Unicast: filtered by the switch. The MAC table delivers the frame to one port only. Other devices never see it.
- Broadcast: destination MAC is all ones (FF:FF:FF:FF:FF:FF), which means everyone. The switch cannot filter it and floods it out every member port of the room. Every device receives it, opens it, and decides at a higher layer whether it is relevant.
- This is why flat networks have a blast radius cost even without security concerns. Every shout burns CPU on every device in the room.

### The two dials on every frame

Every frame has two independent properties. Keeping them separate keeps everything clean.

- Broadcast vs unicast: who the frame is addressed to (everyone vs one MAC)
- Tagged vs untagged: whether the frame is wearing a room label

All four combinations are real. Examples from the lab:

- Untagged unicast: PC loading the ESXi UI (PC to switch to copper to vmnic3 to vSwitch0 to the Management Network outlet where vmk0 is plugged in)
- Untagged broadcast: a phone shouting for a DHCP lease on the flat home network
- Tagged unicast: the DHCP relay forwarding a request to DC01, tagged 10
- Tagged broadcast: a client VM's DHCP shout after being stamped 20 by its port group, flooding room 20 over the trunk

### The DHCP relay round trip (future VLAN 20 to VLAN 10)

Key rule first: a broadcast never crosses a wall. Ever. The relay does not punch a hole. It translates.

Request leg:

1. Client VM in room 20 writes a plain shout (Windows is label blind)
2. Lab-VLAN20 port group stamps tag 20 at the outlet
3. Frame floods room 20 over the trunk, including the switch's own brain, where the relay listens
4. The shout dies at the relay. The relay writes a brand new envelope: unicast, to DC01, from the relay's own room 10 address. Broadcast becomes unicast.
5. The MikroTik stamps tag 10 (frame born in the brain, so the switch stamps)
6. Lab-VLAN10 port group peels the tag
7. DC01 reads a plain envelope and builds the lease

Reply leg:

1. DC01 replies to the relay, not the client (the client has no address yet). Reply is born plain.
2. Lab-VLAN10 port group stamps tag 10
3. Frame terminates at the switch's brain. Tag peeled, envelope opened, frame dies.
4. Relay writes a new envelope to the client with the lease inside
5. MikroTik stamps tag 20
6. Lab-VLAN20 port group peels
7. Client has its lease and never saw a tag

The information crosses the wall. The frame never does. One frame dies at the translator and a second frame is born in the other room.

### The three stampers

The label is always applied by the nearest label aware device on the frame's behalf, and always removed before a label blind device sees it. Tags exist only in the middle of the journey.

- Frame born in a VM or vmk port: the port group stamps it on the way out
- Frame born in the switch's own brain (relay, future gateway): the MikroTik stamps it
- Frame born in a label blind physical box on an access port (future NAS): the switch port stamps it on arrival

### Same room traffic never tags

If two VMs share a room on the same vSwitch, the vSwitch delivers vNIC to vNIC inside the server's RAM. The frame is untagged its entire life. No trunk, no MikroTik, no stamp. Stamping is for travel down the shared cable. Local delivery needs no passport. Side effect: same room same host traffic runs at memory speed and never touches the 10G link.

### What the trunk carries

- Today: tag 10 traffic only when something is plugged into Lab-VLAN10 (currently nothing), plus untagged home network broadcasts, because sfp-sfpplus1 is still a member of the default room. Those broadcasts arrive at vSwitch-Lab, find no untagged outlet, and are dropped. Harmless junk mail, not a leak.
- The copper run is not a trunk. A trunk specifically carries multiple rooms via tags. One cable in the lab is a trunk: the DAC.
- Later: every new VLAN is one more row in the bridge VLAN table with the trunk as a tagged member. Labels 20, 30, 99 all ride the same cable. No new cables ever.
- Hairpinning: once inter VLAN routing exists, traffic between rooms inside the same physical server goes up the trunk in one tag and comes back down the same trunk in the other tag. Neighbors inside one box still send mail out to the translator and back.
- Future hardening item: remove sfp-sfpplus1 from the default room so the trunk carries only tagged traffic. Not urgent, nothing is broken.

### Broadcast flooding is simultaneous, not searching

A shout is copied out every member port of the room at once. Each copy lives or dies independently at its destination. Frames do not wander looking for a home. The router answers DHCP shouts not because the frame ends up there last, but because the router is the DHCP server: the one device in the room whose job is to respond.

### DC01 and the home network, before and after migration

- Today: DC01 sits on VM Network (vSwitch0), which is the flat network. It hears every broadcast from every device in the apartment and ignores them.
- The home router is the DHCP server on the flat network. DC01 is not running DHCP yet.
- Rogue DHCP danger: installing DHCP on DC01 while it sits on the flat network would put two DHCP servers in the same room, racing to answer every device in the apartment. This is exactly why the roadmap order is migrate first, then install DHCP. After migration, DC01's offers can only be heard inside room 10.
- After migration: the copper copy of a home broadcast dies at vSwitch0 (no DC01 outlet there anymore), the trunk copy dies at vSwitch-Lab's door (untagged, no matching outlet). DC01 hears nothing from the apartment. Zero shared broadcast domain.

### The migration: one dropdown, four changes

Editing DC01's vNIC from the VM Network outlet to the Lab-VLAN10 outlet changes, automatically and instantly:

1. Outlet: VM Network to Lab-VLAN10
2. Hallway: vSwitch0 to vSwitch-Lab (port groups live inside vSwitches)
3. Physical exit: vmnic3 copper to vmnic1 DAC (each vSwitch has one uplink)
4. Room and stamping: default room to room 10, stamped and peeled at the outlet

What does not change: anything inside Windows. DC01 keeps its old flat network static IP, its gateway still points at the home router, and everything that points at DC01 for DNS now points at an address in the wrong room. The dropdown is the move. The migration is the checklist: new static IP, DNS updates, gateway, AD health check. Layer 2 changes are instant and silent. The outage they cause lives at layer 3.

### Access ports (future VLAN 30 / TrueNAS)

- For networking purposes the NAS is just a computer plugged into the switch with a cable. Label blind, like Windows.
- In the bridge VLAN table, every port membership is either tagged or untagged. Tagged means the device on the other end understands labels and the cable carries several rooms (the DAC to ESXi). Untagged means the device is label blind and the cable carries one room only.
- An access port is just a port listed as an untagged member of one room. The port peels labels off frames going out to the device and stamps the room's label onto frames coming in from it. The device sees plain envelopes its whole life.
- An access port is the switch doing for a physical box what a port group does for a VM. Same job, different doorway.
- Room 30 will contain exactly two tenants: the ESXi host (a future vmk1 in a Lab-VLAN30 port group) and the NAS. Storage traffic is the most private and performance critical traffic in the lab, which is why it gets its own room. End machines never join room 30.
- Client access to future SMB shares is routed traffic: room 20, through the door at the MikroTik, into room 30. Rules can later restrict that door to file sharing only. This is the Security+ concept of filtering traffic between segments at the boundary.
- Sample trace, ESXi to NAS: frame born untagged at vmk1, Lab-VLAN30 port group stamps 30, rides the trunk as tagged unicast, bridge delivers to the NAS's access port, access port peels, NAS reads a plain envelope. Reply mirrors it: access port stamps 30, port group peels.

## Misconceptions caught (the valuable part)

- Recurring slip all session: granting label awareness to label blind devices. Windows never stamps its own frames. Not DC01, not a client VM, not the NAS. Every time the sender is a familiar machine the brain wants to give it tagging powers. Say the stamper out loud on every trace.
- "The sender tags the frame" - wrong. The nearest label aware device tags on the sender's behalf.
- "Place a rule on VLAN 10 to accept frames tagged 20" - no such rule exists. Broadcasts never cross walls. The relay translates instead.
- "The DC gives out the lease on the home network" - DC01 is not a DHCP server yet. The home router is. Installing DHCP before migrating would create rogue DHCP.
- "Frames ride the trunks until they realize they have nowhere to go" - frames do not search. Flooding is one simultaneous copy to every member port.
- "Copper trunk" - the copper run carries one untagged room. Only the DAC is a trunk.
- Same room traffic got a stamp in my first trace - it never tags at all. Local delivery needs no passport.
- Lesson learned on tracing: before moving any frame, state the map first. Who is talking to whom, and which room does each one live in. Every lost trace this session came from a skipped map, not a broken concept.

## Next session

DC01 migration into VLAN 10, hands on, one step at a time:

1. Pre migration AD health check
2. vNIC dropdown: VM Network to Lab-VLAN10
3. New static IP in room 10, DNS, gateway updates inside Windows
4. Post migration AD health check
5. Then DHCP role scoped to VLAN 10
