# 2026-07-14 - DC01 migration into VLAN 10

The interview-ready one-liner for all of this: _"I migrated a domain controller into a segmented VLAN and had to troubleshoot the full path layer by layer - missing L3 gateway interface, missing routes, missing return path, and a stale gateway config -using the error messages to isolate each layer."_ That is genuinely a junior sysadmin war story now.
## Goal

Move DC01 from the flat home network (VM Network) into VLAN 10 (Lab-VLAN10) with a new static IP, verify DNS self-corrected, and confirm AD health. Completed, with significant unplanned routing troubleshooting along the way.

## Note on addresses

Home network addresses in this note are placeholders in 10.0.0.x form. Lab addresses (10.10.10.x) are real.

## Pre-migration state

- DC01 on VM Network (flat home network), static 10.0.0.10, gateway configured as 10.0.0.1, DNS loopback
- Baseline dcdiag: all structural tests passed. One known cosmetic failure: SystemLog test flagged a SAM/KDC lookup error (EventID 0xC0000007, blank account name, lookup type 0x108), a known noise event on Server 2025. Recorded as baseline so post-migration results could be compared.
- Snapshot taken in ESXi: pre-vlan10-migration

## The migration

1. ESXi: DC01 Network Adapter 1 moved from VM Network to Lab-VLAN10 port group (VM running, via ESXi console throughout, never RDP)
2. New static config on DC01: 10.10.10.10 /24, gateway 10.10.10.1, preferred DNS 10.10.10.10, alternate 127.0.0.1
   - DNS order changed from loopback-only to own-IP-first per Microsoft best practice for DCs

## Troubleshooting: the doorway was never built

Connectivity tests failed after the move. Root causes found and fixed one layer at a time:

### 1. No L3 gateway interface existed

Bridge VLANs table (the sorting rules) had been configured on build night, but no VLAN interface was ever created, so 10.10.10.1 did not exist anywhere. These are two separate things in RouterOS:

- Bridge - VLANs: which ports may carry which VLAN (the walls)
- Interfaces - VLAN: the switch CPU standing inside a VLAN with an address (the door)

Fix: created interface vlan10 (VLAN ID 10, parent interface bridge), assigned 10.10.10.1/24 to it.

### 2. False positive on the earlier gateway test

A ping to 10.10.10.1 from the home PC had returned replies before the interface existed. The reply came from ISP infrastructure upstream (10.x.x.x is used inside carrier networks). Lesson: a ping reply proves someone answered, not that the right someone answered.

### 3. Switch had no presence on the home network

The MikroTik had only its factory default 192.168.88.1 and the new vlan10 address. No home network address, no default route (this is also why WinBox only worked via MAC). A router can only route between networks it stands in.

Fix: added IP - DHCP Client on bridge. Switch received 10.0.0.110 dynamically plus a default route to the home router.

### 4. No return path from the home network to the lab

The home router has no route for 10.10.10.0/24, so replies to lab traffic would be lost.

Fix (pragmatic): srcnat masquerade rule on the MikroTik - chain srcnat, src address 10.10.10.0/24, out interface bridge, action masquerade. Lab traffic leaving toward the home network is re-addressed as the switch itself. The textbook alternative (static route on the home router) is parked for later.

### 5. Ghost gateway discovered

DC01's original config pointed at gateway 10.0.0.1. The home router actually lives at 10.0.0.254. The wrong value sat silent for months because a host on a flat network rarely exercises its gateway for local traffic. First real knock on the door tonight revealed no house there. Lesson: wrong settings can hide until an architecture change exercises them.

## Post-migration verification

- ping 10.10.10.1: replies (gateway reachable inside VLAN 10)
- ping home router at 10.0.0.254: replies (crossing the door works)
- ping 8.8.8.8: replies, ~12ms (full chain to internet works)
- ipconfig /registerdns, netlogon restart, dcdiag /fix run on DC01
- DNS Manager check: landon.lab zone already clean, both Host (A) records showing 10.10.10.10 only, old records replaced not duplicated. SRV records point at the hostname, so they self-corrected.
- nslookup landon.lab: single answer, 10.10.10.10
- dcdiag /q: completely silent (even the baseline cosmetic event did not recur)
- Snapshot pre-vlan10-migration deleted after verification

## End state

- DC01: 10.10.10.10 in VLAN 10, healthy, internet-capable, DNS clean
- MikroTik: routing between VLAN 10 and the home network, NAT masquerade for lab-outbound traffic, dynamic home address via DHCP client
- VLAN 10 is no longer an island: it has a working door

## Lessons

- Bridge VLAN table and VLAN interface are separate: walls vs door
- A ping reply proves someone answered, not that the right someone answered
- A router must stand in both networks to route between them
- Wrong settings can sit silent until something finally exercises them
- Read who a ping error comes from: the local host, the gateway, or beyond - each points at a different layer

## Parked items

- Rename DC01: Windows hostname is still the install-generated WIN-xxxx name. Renaming a DC post-promotion is its own careful procedure. Do as a dedicated session.
- Replace NAT masquerade with a proper static route on the home router (if the ISP box supports it)
- MikroTik home address is a DHCP lease and could change. Consider a reservation on the home router or a static address later. Nothing breaks if it changes except WinBox-by-IP convenience.
- Next roadmap step: DHCP role on DC01 scoped to VLAN 10, then VLAN 20 with DHCP relay

## Supporting Images

![[DC01-to-Internet.png]]