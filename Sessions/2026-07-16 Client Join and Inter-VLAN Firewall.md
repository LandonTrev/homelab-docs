# Session - Client Join and Inter-VLAN Firewall

**Date:** 2026-07-16
**Goal:** Join CLIENT01 to the domain, create the first domain users, and build inter-VLAN firewall rules so VLAN 20 (clients) can only reach the AD services it needs on VLAN 10 (servers).

## What happened

### Domain join
- CLIENT01 already had a working DHCP lease from the relay (10.0.0.x in VLAN 20, DNS pointing at DC01).
- Joined CLIENT01 to landon.lab via System Properties (sysdm.cpl) - Change - Domain.
- Computer account appeared in the default Computers container in ADUC. (Moving it to a proper OU is a later Group Policy exercise.)

### Machine rename
- Default name was DESKTOP-XXXXXX. Renamed to CLIENT01.
- Rename done ON THE CLIENT, not from ADUC (see Gotchas).
- First `Rename-Computer` attempt failed with "Access is denied" while logged in as a local account. Fixed by logging in as the domain admin and running the rename without -DomainCredential.

### User accounts
- Created first standard domain user: ltrevisani (convention: first initial + last name). This is now the account naming standard for landon.lab.
- Created ltrevisani-adm as a separate admin account and added it to Domain Admins.
- Standard user for daily work, -adm account only for elevation / DC tasks. Built-in Administrator to be retired from daily use later.

### Inter-VLAN firewall (the main event)
- Built four rules in the MikroTik forward chain (IP - Firewall - Filter Rules).
- Order matters (first match wins); final order is:
  1. accept established/related
  2. accept AD services TCP (20 -> 10)
  3. accept AD services UDP (20 -> 10)
  4. drop all other (20 -> 10)
- Full rule detail and reasoning: see [[Topics/Firewall-Rules-Concepts]] and [[Topics/AD-Ports-and-Services]].

## Testing
- From CLIENT01 as ltrevisani:
  - `nslookup landon.lab` returned 10.0.0.x (DC address). DNS through the wall works.
  - `ping 10.0.0.x` (DC) timed out. This is the GOOD result - ICMP is not an allowed door, so the drop rule caught it. A failed ping here proves the wall works.
- Both halves confirmed: an unlocked door (DNS) works, an unlocked-for door (ping) is blocked.

## Gotchas captured
- ADUC does not show up reliably in Start search on Server 2025. Launch with `dsa.msc`, or Server Manager - Tools - Active Directory Users and Computers. The "Active Directory Module for Windows PowerShell" entry is NOT ADUC - it just opens a PowerShell console. If dsa.msc is missing entirely: `Install-WindowsFeature RSAT-ADDS-Tools`.
- NEVER rename a domain-joined machine from ADUC. Renaming the object only relabels it in AD; the machine still calls itself the old name and the two disagree, breaking trust. Rename on the machine itself so it updates its own AD object and DNS.
- "Access is denied" on Rename-Computer: happens when logged in as a local account juggling -DomainCredential. Fix: log in AS the domain admin, then rename with no -DomainCredential (domain admins are automatically local admins on member machines).
- AD requires the user object Full Name (CN) to be unique within a container. Two users with the same Full Name collide even with different logon names. Adjusted the admin account Full Name to "Landon Trevisani - Admin" to resolve. Bonus: makes admin accounts easy to spot in ADUC.

## Next
- Confirm a FRESH login as ltrevisani still works (sign out and back in), to prove Kerberos (port 88) flows through the wall and not just a cached login.
- Inter-VLAN rules for the reverse direction (10 -> 20) as a separate, later decision.
- DC rename (netdom) still parked - must happen before any PKI / AD CS work.
