# ESXi on Hybrid CPUs (the MS-01 Purple Screen)

The problem: Intel 12th and 13th gen mobile CPUs (like the i9-13900H) mix Performance and Efficiency cores. ESXi expects every core to be identical, so its boot-time uniformity check panics with a purple diagnostic screen:

```
HW feature incompatibility detected; cannot start
Fatal CPU mismatch on feature "Hyperthreads per core"
Fatal CPU mismatch on feature "Cores per package"
```

This is a platform behavior, not broken hardware. Standard on the MS-01 with ESXi 8.

## Fix

One-time boot (installer, or before the permanent fix is set):
At the ESXi boot screen press Shift+O, then append to the boot line:

```
cpuUniformityHardCheckPanic=FALSE
```

Permanent (survives every reboot) - from the ESXi Shell (DCUI, F2, Troubleshooting Options, Enable ESXi Shell, then Alt+F1):

```
esxcli system settings kernel set -s cpuUniformityHardCheckPanic -v FALSE
```

Verify - Configured must read FALSE:

```
esxcli system settings kernel list -o cpuUniformityHardCheckPanic
```

## Notes
- The installer flag does not persist - expect one more purple screen on first boot after install; apply the flag once more, then set the permanent fix.
- The flag downgrades the hard-check panic so the kernel boots; the scheduler treats cores as capable enough. Fine for a homelab; unsupported config for production.
- "settings" is plural in the command - "system setting" errors with Unknown command.

Learned in: [[Sessions/2026-07-08 ESXi Install]]
