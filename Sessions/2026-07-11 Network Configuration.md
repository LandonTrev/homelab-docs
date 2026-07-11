# Session: ESXi Network Configuration

**Date:** 2026-07-11
**Goal:** Get ESXi host on the network, access web UI

## What happened
- Plugged router Ethernet into a 2.5GbE RJ45 port on the MS-01.
- Host showed `169.254.x.x` (APIPA) + "Waiting for DHCP" → DHCP lookup failed.
- Root cause: management network (vmk0) was bound to the wrong vmnic.

## Diagnosis
- DCUI → F2 → Configure Management Network → Network Adapters
- Found the bound NIC (vmnic1) was **Disconnected**.
- Two NICs showed **Connected**: the router port AND the SFP+ DAC to the MikroTik.
  - The DAC reports link-up even with the MikroTik unconfigured, because a
    direct point-to-point copper link electrically detects both ends - no
    switch config required. But there's no DHCP server behind it, so binding
    to it leaves the host on 169.254.

## Fix
- Selected **only vmnic3** (the connected router port), deselected all others.
- Did NOT team both NICs — one uplink for management.
- Enter → Esc → Y to restart management network.
- Host pulled a real `192.168.x.x` lease.
- Accessed web UI at `https://<IP>/ui` ✓

## Key takeaways
- "Connected" in DCUI = physical link only, NOT a working management path.
- Use **D (View Details)** on a vmnic to confirm link speed:
  DAC ≈ 10000 Mbps, 2.5GbE router port ≈ 2500 Mbps — tells you which is which.
- Leave the DAC physically connected; it's needed later for the 10GbE
  storage backbone (NFS/iSCSI from TrueNAS). ESXi just isn't using it for
  management right now.

## Next
- Build first VM: Windows Server AD Domain Controller.