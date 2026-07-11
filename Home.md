# Home

Enterprise-simulating home lab: VMware ESXi, Active Directory, TrueNAS shared storage, 10GbE MikroTik backbone. Built from the ground up and documented as I go.

End goal: a small business environment on ESXi - an AD domain built from scratch (domain controller, DNS/DHCP, Group Policy), TrueNAS providing NFS/iSCSI datastores and AD-integrated SMB shares, all on a 10GbE backbone.

Links: [[Hardware]] | [[Roadmap]] | [[Sessions/2026-07-08 ESXi Install|Latest session]]

---

## Status

### Done
- [x] ESXi 8.0.3 installed on MS-01 (Fanxiang 256GB boot drive)
- [x] i9-13900H purple-screen fix made permanent (cpuUniformityHardCheckPanic=FALSE)

### Next up
- [ ] Network the host - 2.5GbE copper direct to router, get DHCP, reach web UI at https://IP/ui
- [ ] Apply free license in web UI
- [ ] Create VM datastore on Samsung 990 PRO 2TB
- [ ] Configure MikroTik CRS310 (RouterOS, uplink to router)
- [ ] TrueNAS on the DXP2800 - ZFS mirror, then NFS/iSCSI to ESXi
- [ ] First VM: Windows Server, then AD Domain Controller
- [ ] AD services: DNS/DHCP, GPOs, AD-integrated SMB shares

---

## How this vault works
- Hardware: every device, specs, quirks. Reference; rarely changes.
- Roadmap: certs, lab phases, career. Updated at milestones.
- Sessions/: one dated note per lab day. Raw history, never edited after.
- Topics/: distilled concept notes (what a thing is and how my lab uses it). The polished, public-facing layer. Sessions feed Topics.

New session note - copy this:

```
# YYYY-MM-DD - Topic

Goal:
Result:

## What happened
-

## Lessons
-

## Next time
- [ ]
```
