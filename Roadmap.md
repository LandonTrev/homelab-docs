# Roadmap

Three tracks in parallel: certs, lab, career. Updated at milestones.

---

## Certifications
1. Security+ (SY0-701) - IN PROGRESS. Messer videos, study guide, Dion practice tests. Gate: 85% or higher on practice exams before booking.
2. Cloud fundamentals - AZ-900 or AWS Cloud Practitioner
3. Server+ (optional, time permitting)
4. CySA+ - ruled out unless committing to a SOC/analyst direction

## Lab phases
1. Hypervisor - ESXi 8 stable on MS-01 (DONE)
2. Network - host on the LAN (copper interim, then CRS310 configured with router uplink)
3. Storage - TrueNAS ZFS mirror, NFS/iSCSI datastores to ESXi
4. Identity - Windows Server VM, AD domain controller built from scratch
5. Services - DNS/DHCP, Group Policy, AD-integrated SMB shares with proper permissions
6. Later: VLAN segmentation on the MikroTik, more member servers and clients

## Career
- [ ] Job description mapping - pull 5 to 10 real Junior Sysadmin and entry cyber postings; tally repeated must-haves vs nice-to-haves; watch for clearance language and DoD 8140/8570 IAT II; refine this roadmap against what postings actually ask for
- [ ] Publish this vault's Topics, Hardware, and Roadmap to GitHub as the public knowledge base
- [ ] LinkedIn About: update as milestones land (planned becomes running)
- [ ] Resume bullets from completed lab phases

## Principles
- One deliberate step at a time; confirm before destructive actions
- Foundations in order: hypervisor, network, storage, VMs
- Every lab day gets a session note; recurring concepts get promoted to Topics
