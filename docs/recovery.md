# Recovery and rollback

## Boot the preserved Intel image

Select **Omarchy Intel recovery** in Limine. The image created during this repair contains the working Intel-only command line, including `module_blacklist=amdgpu`. It is separate from the normal AMD-enabled image.

This static image is a same-kernel recovery option, not a replacement for snapshots or installation media. After a kernel upgrade, modules on the root filesystem may no longer match it. Refresh and verify recovery images as part of any later kernel investigation.

## Temporary Intel-only boot

If needed, edit the normal T2 entry with **E** and append:

```text
apple_gmux.force_igd=1 module_blacklist=amdgpu
```

Then press **F10**. An external keyboard may be necessary for that key. Preserve the existing encrypted-root parameters. The blacklist affects only that boot unless you write it to persistent configuration.

## Make Intel-only operation persistent again

For this repository's exact installed layout, replace `/etc/limine-entry-tool.d/zz-local-t2-graphics.conf` with:

```bash
KERNEL_CMDLINE[default]+=" apple_gmux.force_igd=1 module_blacklist=amdgpu video=Unknown-1:d plymouth.enable=0"
```

Then rebuild and reboot:

```bash
sudo limine-mkinitcpio linux-t2
sudo reboot
```

This fallback retains text disk unlock and the firmware-output exclusion. The original verified Intel recovery image predates those two settings; it remains a separate alternative. AMD acceleration and external outputs wired through AMD are unavailable while the driver is blocked.

The gmux/i915 early-loading configuration can remain. The AMD performance rule has no effect while AMD is blocked.

## Restore the pre-fix configuration

Use the backup taken before installation. Restore the original versions of the affected files; remove only files that this procedure created and that did not previously exist:

```text
/etc/modprobe.d/apple-gmux.conf
/etc/mkinitcpio.conf.d/zz-local-t2-graphics.conf
/etc/mkinitcpio.conf.d/zz-local-t2-amd-power.conf
/etc/udev/rules.d/99-local-t2-amd.rules
/etc/limine-entry-tool.d/zz-local-t2-graphics.conf
```

Run `sudo limine-mkinitcpio linux-t2` after restoring configuration. Restoring the pre-fix state may restore the original failure as well, so prefer the Intel recovery option when the goal is simply to regain a usable machine.

The original repair session stored backups under `/root/t2-graphics-backup-20260914` and `/root/t2-amd-test-20260914`. The reproducible README procedure instead uses `/root/t2-graphics-before`; use the directory that actually exists on your machine.

## One-time test selection

On the tested Limine version, `bootctl list` reported the custom test entry ID as `Omarchy-AMD-test`. This selected it once without changing the permanent default:

```bash
sudo bootctl set-oneshot Omarchy-AMD-test
sudo reboot
```

The test entry intentionally starts in text mode. After checking driver binding and clocks, the desktop can be started with `sudo systemctl start sddm`. Do not use a displayed title as a guessed one-shot ID; inspect `bootctl list` first.

## If Hyprland says safe mode

A compositor crash can cause `start-hyprland` to relaunch with `--safe-mode`. This happened during the rejected live phantom-monitor approach; it was a separate failure from the original power-off.

Restore the last working Hyprland configuration before restarting the session. A reload in safe mode is not proof that the user configuration has been loaded. Check the process arguments and `hyprctl configerrors` after returning to a normal session. Restarting SDDM ends graphical applications, so save any surviving work first.
