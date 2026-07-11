# 2026-07-08 - ESXi Install on MS-01

Goal: First boot, verify hardware in BIOS, install ESXi 8.
Result: ESXi 8.0.3 installed to the Fanxiang 256GB; CPU-panic fix made permanent. Host not networked yet (expected - switch unconfigured).

## What happened
- BIOS checks: RAM at 5200 on 5600 sticks (normal; BIOS max already set to 5600). Could not find an NVMe drive list - spent a while hunting before confirming this BIOS has no drive list at all. VMD already disabled, all PCIe root ports enabled, boot menu showed generic "UEFI NVME" which means drives are detected. Blank drives never get named boot entries.
- USB failure: the planned SanDisk 128GB stick reported 64MB. diskpart clean succeeded but capacity stayed 64MB - controller-level, not partitioning. Flash Drive Info Extractor confirmed: Physical Disk Capacity = 67,108,864 bytes (exactly 64MB), placeholder serial, controller unidentifiable. Unrecoverable - swapped to a Sony 8GB stick, worked first try.
- ISO: free ESXi lives on support.broadcom.com now (free account, product name "VMware vSphere Hypervisor"). Downloaded VMware-VMvisor-Installer-8.0U3e-24677879.x86_64.iso (about 618MB) - newest free build; later patches (3g/3h/3i) are paid-only.
- Rufus: GPT, UEFI (non-CSM), "Write in ISO Image mode." A transient format warning mid-write; result verified healthy.
- Install: immediate purple screen - "Fatal CPU mismatch" (hybrid P/E cores, known MS-01 issue). Shift+O at installer boot, append cpuUniformityHardCheckPanic=FALSE, booted clean. Disk screen showed both NVMe drives by name. Installed to the Fanxiang; 990 PRO untouched.
- First boot purple-screened again (installer flag does not persist). Shift+O once more, reached DCUI, enabled ESXi Shell (F2, Troubleshooting Options), Alt+F1, then ran:
  esxcli system settings kernel set -s cpuUniformityHardCheckPanic -v FALSE
  Verified with: esxcli system settings kernel list -o cpuUniformityHardCheckPanic - Configured: FALSE. Permanent.
- DCUI shows "DHCP lookup failed / 0.0.0.0" - the SFP+ path dead-ends at the unconfigured MikroTik. Known issue, next session.

## Lessons
- "Drives not detected" is not the same as "I cannot find where the BIOS lists drives." Confirm the problem is real before tearing hardware apart - the ESXi installer disk screen is the ground truth on this box.
- A drive that survives diskpart clean but still reports wrong capacity is hardware, not config - partitions live above what the controller advertises. Verify new flash media (H2testw or ValiDrive) on day one, and time-box recovery attempts on consumables.
- esxcli system settings - plural; the singular form errors out.
- Vendor GB vs OS GiB: 64GB RAM shows as 63.7 GiB, 2TB as 1.82 TiB. Nothing is missing.

## Next time
- [ ] Copper to router, DHCP, web UI
- [ ] Free license, datastore on the 990 PRO

Related: [[Topics/ESXi on Hybrid CPUs]] | [[Topics/ESXi Basics]]
