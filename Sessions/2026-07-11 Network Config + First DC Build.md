# Session — ESXi Networking Fix + First Domain Controller Build

**Date:** 2026-07-11 **Goal:** Get ESXi on the network, access the web UI, build the first VM as an AD Domain Controller **Outcome:**  Domain `landon.lab` is live on DC01

---

## 1. ESXi Management Networking

### Problem

Host showed `169.254.184.16` (APIPA) with "Waiting for DHCP..." and a DHCP lookup failed warning. Web UI unreachable from laptop.

### Diagnosis

DCUI → **F2** → Configure Management Network → Network Adapters

|Device|Status|
|---|---|
|vmnic0|Disconnected|
|vmnic1|**Connected (...)** ← was bound [X], but not the router port|
|vmnic2|Disconnected|
|vmnic3|**Connected** ← actual router uplink|

Management (vmk0) was bound to the wrong NIC.

**Why two NICs showed "Connected":** one was the 2.5GbE router port, the other was the SFP+ DAC to the MikroTik. The DAC reports link-up even though the MikroTik is unconfigured — a direct point-to-point copper link electrically detects both ends without any switch config. But no DHCP server sits behind it, hence the 169.254 fallback.

### Fix

- Deselected vmnic1, selected **only vmnic3**
- Did NOT team both NICs (one uplink for management)
- Enter → Esc → **Y** to restart management network
- Host pulled `192.168.254.103` ✓
- Web UI reachable at `https://192.168.254.103/ui`

---

## 2. Storage — Datastore Provisioning

Initial state: only one datastore (`datastore1`, 110.25 GB) — that was the **Fanxiang 256GB boot drive**. The Samsung 990 PRO 2TB was installed but never formatted (intentionally left untouched during ESXi install).

**Actions:**

- Renamed boot datastore → **`ESXiBoot`** (110.25 GB)
- Created new datastore → **`vmstore`** on the Samsung 990 PRO — **1.82 TB, VMFS6**
- Created `vmstore/ISOs/` directory
- Uploaded Windows Server 2025 eval ISO (~7.7 GB) — build `26100.32230.260111-0550`

`vmstore` is now the VM home. `ESXiBoot` holds ESXi only.

---

## 3. VM Build — DC01

### OS choice: Windows Server 2025 Standard (Desktop Experience)

Rationale:

- Server 2022 mainstream support **ends October 2026** — a fresh 2022 build enters maintenance mode almost immediately
- Most orgs upgrading are going straight to 2025 rather than stopping at 2022
- 2025 changed AD security defaults in ways directly relevant to the security track: **Credential Guard on by default**, **NTLM deprecated in favor of Kerberos**, **SMB signing required by default**
- 180-day eval, no license needed for lab

Standard (not Datacenter) — Datacenter only matters for unlimited VM licensing / Storage Replica at scale. Desktop Experience (not Core) — GUI while learning AD. Core is a separate future exercise.

### VM specs

|Resource|Value|
|---|---|
|Name|`DC01`|
|vCPU|2|
|RAM|4 GB|
|Disk|60 GB, **thin provisioned**|
|Datastore|`vmstore`|
|NIC|VMXNET3 → `VM Network`|
|Firmware|EFI|

---

## 4. AD DS Install + DC Promotion

- Server Manager → Manage → Add Roles and Features → **Active Directory Domain Services**
- Promoted to DC: **Add a new forest**
- **Root domain:** `landon.lab`
- **NetBIOS:** `LANDON`
- DNS Server role installed automatically (AD depends on DNS)
- Global Catalog: yes (forced — first DC in forest)
- **DSRM password set** — stored separately, NOT in these notes

### Prereq check warnings (both expected)

1. **No static IP** — legitimate, fixed immediately after promotion (see below)
2. **DNS delegation cannot be created** — expected. No parent zone exists for `.lab` because it isn't a real TLD. Wizard itself says "no action is required."

Server rebooted automatically. Login became `LANDON\Administrator`.

---

## 5. Static IP (post-promotion)

A DC **must** have a static IP. It's also the DNS server for the domain — if the address shifts on a DHCP lease renewal, every client loses name resolution and the domain effectively breaks. No exceptions in production.

`ncpa.cpl` → Ethernet0 → IPv4 Properties:

|Field|Value|
|---|---|
|IP address|`192.168.254.10`|
|Subnet mask|`255.255.255.0`|
|Default gateway|`192.168.254.1`|
|Preferred DNS|`127.0.0.1`|

**DNS points at loopback, not the router.** The DC _is_ the domain's DNS server — it resolves against itself. Pointing it at the router would break AD name resolution.

Chose `.10` to sit outside the router's DHCP pool (pools typically start at `.100`).

### Verification

```
ipconfig /all
  DHCP Enabled . . . . . . . . . . : No
  IPv4 Address . . . . . . . . . . : 192.168.254.10 (Preferred)
  Default Gateway  . . . . . . . . : 192.168.254.1
  DNS Servers  . . . . . . . . . . : ::1
                                     127.0.0.1
  Primary DNS Suffix . . . . . . . : landon.lab
  Description  . . . . . . . . . . : vmxnet3 Ethernet Adapter

nslookup landon.lab
  Name:    landon.lab
  Address: 192.168.254.10
```

`nslookup` resolving the domain back to the DC's own IP confirms AD created its DNS zone and the DC registered itself. **Domain is live.**

---

## Key takeaways

- **"Connected" in the ESXi DCUI means physical link only** — not a working uplink. An SFP+ DAC shows link-up even with the switch on the other end completely unconfigured, because point-to-point copper detects both ends electrically. Use **`D` (View Details)** on a vmnic to check link speed and disambiguate: DAC ≈ 10000 Mbps, 2.5GbE copper ≈ 2500 Mbps.
- **Creating a VM builds hardware, not an OS.** A fresh `.vmdk` is a blank disk — "No Media" on the virtual disk at first boot is _expected_. The ISO is the installer that fills it. Once Windows is installed, the ISO becomes irrelevant and the VM boots from disk.
- **A VM's folder on the datastore IS the VM.** `DC01.vmx` (config), `DC01-flat.vmdk` (disk data), `DC01.nvram` (UEFI state), `vmware.log`. Copy the folder, register the `.vmx`, and the VM comes back elsewhere. That's file-level migration/backup.
- **Edit settings changes don't take effect until Save is clicked.** Cost ~20 minutes of "No Media" debugging when the config screen showed correct settings that had never been committed.
- **Domain controllers get static IPs, and point DNS at themselves.** Non-negotiable.
- **Don't put AD CS (PKI) on a domain controller.** Best practice is a dedicated member server, and a real design uses a two-tier PKI (offline root CA + issuing subordinate). Build the domain first, then the PKI on top.

---

## Next session

- Create OUs, users, and groups in AD
- Build a client VM and join it to `landon.lab`

## Parked / roadmap

- **Remote lab access:** Tailscale as a subnet router. Run it on **TrueNAS**, _not_ the MS-01 — if the remote-access node lives on the only ESXi host and that host goes down, there's no way in to fix it. Chicken-and-egg lockout. Router-based WireGuard is the more "enterprise" alternative and better GitHub material.
- **AD CS / PKI:** separate member server, later. Strong project for the gov/defense track (smartcard/CAC auth, LDAPS).
- **Azure Arc:** declined the first-login prompt. Worth revisiting as an AZ-900-adjacent exercise once there's a free Azure account. Also the mechanism behind Server 2025 hotpatching.
- **Server Core:** practice a headless install as a separate exercise.