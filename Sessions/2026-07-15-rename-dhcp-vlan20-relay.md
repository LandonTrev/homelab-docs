# 2026-07-15 - DC rename, DHCP, and VLAN 20 with DHCP relay

## Goal

Three phases in one session: rename DC01 properly, stand up the DHCP role scoped to VLAN 10, then build VLAN 20 (clients) end to end including an inter-VLAN DHCP relay. Proven working by a real Windows 11 client pulling a lease across the VLAN boundary.

## Note on addresses

Lab addresses (10.10.10.x, 10.10.20.x) are real. Home network addresses are placeholders in 10.0.0.x form.

## Phase 1 - DC01 rename

Background: the machine was promoted to a DC while still using its factory-generated name (WIN-JGRU73KEUDN). "DC01" was only a spoken label. Renaming now, before DHCP and any future PKI, keeps the change cheap since nothing depends on the name yet.

Steps:
1. Snapshot pre-rename in ESXi.
2. Confirmed current name with hostname (WIN-JGRU73KEUDN).
3. netdom computername WIN-JGRU73KEUDN /add:DC01.landon.lab (adds the new name as an alternate, both names work).
4. netdom computername WIN-JGRU73KEUDN /makeprimary:DC01.landon.lab (promotes DC01 to primary).
5. Restart-Computer. Hostname confirmed DC01 after reboot.

DNS cleanup (the automatic cleanup partially failed):
- ipconfig /registerdns re-registered records under the new name. SRV records in _tcp self-corrected to dc01.landon.lab.
- Manually deleted the stale win-jgru73keudn Host (A) record in the landon.lab zone.
- The _msdcs NS record still pointed at the old name. NS records are edited via the zone's Name Servers tab, not deleted. Edited it to dc01.landon.lab. (Lesson: NS records are replaced, not removed, or the zone loses a listed nameserver.)

Verification:
- dcdiag /q kept replaying historical SystemLog events (Group Policy timing, the failed DNS deletion, a DCOM reach to the old home router). These were frozen log entries from the reboot window, not live faults.
- dcdiag /test:dns passed. repadmin /showrepl clean, DC identifying as DC01 with a valid GUID.
- Cleared the System event log and re-ran dcdiag /q: came back clean, confirming no new errors were being generated. (Lesson: clearing the log separates "already happened" from "happening now.")

## Phase 2 - DHCP role on DC01, scoped to VLAN 10

1. Snapshot pre-dhcp.
2. Add Roles and Features wizard: selected DHCP Server role, added its management tools, nothing extra on the Features page, no auto-restart.
3. Completed post-install "Complete DHCP configuration": authorized the server in AD using the current admin credentials, created the two default security groups. (Concept: in an AD domain a DHCP server must be authorized before it can hand out addresses. Prevents rogue DHCP servers.)
4. New scope VLAN10-Servers:
   - Range 10.10.10.100 - 10.10.10.200, mask 255.255.255.0
   - No exclusions (static devices sit below .100: gateway .1, DC .10)
   - Lease 8 days (default; servers are stable)
   - Router 10.10.10.1, DNS 10.10.10.10, DNS domain landon.lab, WINS blank
   - Activated

Note: the wizard auto-populates the DNS server (10.10.10.10) and typing it again produces a duplicate. Removed the extra so the list shows a single entry. Green down-arrow on the server node confirms AD authorization.

## Phase 3 - VLAN 20 (clients) with DHCP relay

Subnet scheme: 10.10.20.0/24, gateway 10.10.20.1, pool 10.10.20.100-200 (third octet matches VLAN number for consistency).

MikroTik (WinBox):
1. Bridge > VLANs: added VLAN 20, tagged on bridge and sfp-sfpplus1. (Walls. VLAN filtering already enabled from the VLAN 10 build, so it enforces immediately.)
2. Interfaces > VLAN: created vlan20 (VLAN ID 20, interface bridge). Then IP > Addresses: 10.10.20.1/24 on vlan20. (Door / gateway. Possible because the bridge CPU is a VLAN 20 member from the tagging above.)
3. IP > Firewall > NAT: srcnat, src 10.10.20.0/24, out bridge, action masquerade (mirror of the VLAN 10 rule; gives VLAN 20 an internet path).
4. IP > DHCP Relay: relay-vlan20, interface vlan20, DHCP server 10.10.10.10, local address 10.10.20.1.

ESXi:
- Created port group Lab-VLAN20 on vSwitch-Lab, VLAN ID 20.

DC01:
- New scope VLAN20-Clients: range 10.10.20.100-200, lease 4 days (clients churn more than servers), router 10.10.20.1, DNS 10.10.10.10 (still the DC, cross-VLAN), DNS domain landon.lab.

## Phase 3 test - Windows 11 client (CLIENT01)

Built a Windows 11 Enterprise eval VM as the first lab client.

- Naming: CLIENT01 for both the VM and the Windows computer name (kept matched to avoid a repeat of the DC name mismatch). Convention settled: servers by role (DC01), clients as CLIENTxx. OS type tracked in docs, not encoded in the name.
- vTPM not available: standalone free ESXi cannot use Native Key Provider (requires a vCenter cluster), and the PowerCLI workaround needs a paid Enterprise Plus license. Licensing ESXi purely for vTPM would be a bad trade (enterprise per-core subscription, not a homelab purchase).
- Used the LabConfig registry bypass at the "This PC can't run Windows 11" screen: Shift+F10 > regedit > HKLM\SYSTEM\Setup\LabConfig (new key) > DWORDs BypassTPMCheck=1 and BypassSecureBootCheck=1, then Back and forward to re-run the check. (Initial slip: values were placed directly under Setup instead of inside a LabConfig subkey, so setup ignored them.)
- Network adapter on Lab-VLAN20.
- Privacy toggles set to off/minimum at OOBE, local account (labadmin), no Microsoft account.

Result: ipconfig /all showed 10.10.20.100, gateway 10.10.20.1, DHCP Server 10.10.10.10. The DHCP Server line reading 10.10.10.10 confirms the lease came from DC01 in VLAN 10 across the boundary via the relay.

## Key concepts locked in this session

- DHCP relay round trip: a booting client broadcasts (Layer 2, to ff:ff:ff:ff:ff:ff) because it has no IP yet. The broadcast is bounded by the VLAN and cannot reach DC01 in another VLAN. The relay catches it and sends a directed (unicast) request to DC01, writing giaddr = 10.10.20.1 inside the DHCP message.
- giaddr does two jobs from one field: tells DC01 which scope to use (by which subnet the address falls in) and where to send the reply (back to the relay). DC01 reads it per request and matches a scope; it does not store giaddr.
- Relay is not NAT. Relay uses giaddr for the return path. NAT masquerade (separate rule) rewrites source addresses for internet traffic so a router that does not know the private subnet can still return replies. Both involve addresses but are different mechanisms.
- VLAN tagging vs giaddr are two different stamps at two layers. Tags (L2) are applied fresh per trunk leg by whoever puts the frame on the trunk (port group on the server side, MikroTik interface on the switch side) and stripped on exit. giaddr (L3) is a field inside the DHCP payload.
- Gateway = the address a device sends anything it cannot deliver locally. Always a router interface standing in multiple networks. VMs live on the ESXi side of the trunk; their gateways live on the MikroTik side; traffic crosses the trunk to reach its own gateway.
- Broadcast domain (L2, defined by the VLAN) vs .255 broadcast address (L3, defined by the subnet mask): typically 1:1 with subnet and VLAN by design, but independently defined.
- One bridge with VLANs is the correct segmentation model, not multiple bridges. VLANs give separation plus flexibility.

## Parked / next session

- Domain-join CLIENT01 to landon.lab (the natural next step now that it has an address).
- BitLocker + AD recovery-key escrow on physical hardware (deferred from the vTPM dead-end; do it on a real machine with a real TPM, which is more resume-relevant anyway).
- Native Key Provider / vTPM remains out of reach on standalone free ESXi; revisit only if the host is ever put under vCenter.
- DHCP reservation to pin the MikroTik's home address: deferred until after the apartment move, since that network gets replaced.
- Remaining VLAN roadmap: VLAN 30 (storage/TrueNAS), VLAN 99 (management).
- Snapshots pre-rename and pre-dhcp can be deleted once comfortable everything is stable.
