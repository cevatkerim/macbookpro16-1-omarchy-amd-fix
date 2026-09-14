# Working AMD graphics on a T2 MacBook Pro with Omarchy

This documents a tested configuration for a **2019 16-inch MacBook Pro (`MacBookPro16,1`) with a Radeon Pro 5300M** that initially went black after disk unlock, briefly ran its fans at high speed, and powered off.

The configuration keeps **Intel driving the internal panel and AMD available for accelerated applications**. It enables AMD power management, forces the `low` performance level at device initialization, disables runtime GPU power management, and excludes two phantom display outputs before the desktop starts. It also uses a text disk-unlock prompt instead of Plymouth.

This is a working configuration workaround, not an upstream driver patch or proof that every individual setting is necessary. See [the investigation](docs/investigation.md) and [validation results](docs/validation.md) for what was actually established.

## Tested hardware and software

| Component | Tested configuration |
| --- | --- |
| Machine | MacBook Pro 16-inch, 2019; `MacBookPro16,1` |
| Integrated GPU | Intel UHD Graphics 630, PCI `0000:00:02.0` |
| Discrete GPU | AMD Radeon Pro 5300M / Navi 14, PCI `0000:03:00.0`, ID `1002:7340` |
| AMD memory | 4 GB |
| Internal display | `eDP-1`, 3072×1920 at 60 Hz, scale 2 |
| Omarchy at diagnosis | `4.0.3-1`; installation was originally reported as 4.0.2 |
| Kernel | `7.2.4-arch1-Watanare-T2-2-t2` (`linux-t2` package `7.2.4.arch1-2`) |
| Hyprland | `0.56.2-2` |
| Mesa | `1:26.2.2-1` |
| AMD firmware package | `linux-firmware-amdgpu 20260810-2` |
| Bootloader | Limine 12.8.0, unified kernel image (UKI), encrypted root |
| Validation date | 2026-09-14 |

The machine was initially described as a 15-inch model, but its DMI identifier and GPU subsystem identified it as the 16-inch model above. **Do not assume these settings apply to a 15-inch Polaris/Vega model, an Intel-only 13-inch model, or Apple Silicon.**

## What the configuration changes

| Setting | Purpose |
| --- | --- |
| `apple_gmux.force_igd=1` and the gmux module option | Route the internal panel through Intel |
| Early `apple_gmux` and `i915` modules | Establish Intel display routing during initramfs startup |
| `amdgpu.dpm=1` | Keep AMD dynamic power management support enabled; disabling it failed on this machine |
| `amdgpu.runpm=0` | Prevent runtime suspend/power-off of the AMD device |
| udev performance level `low` | Select the fixed low-performance policy when AMD creates its DRM device |
| Include the udev rule in the initramfs | Apply the policy during early initialization, not just after root is mounted |
| `video=eDP-2:d` | Disable AMD's unusable internal-panel connector while Intel owns the panel |
| `video=Unknown-1:d` | Disable the leftover firmware-framebuffer output |
| `plymouth.enable=0` | Avoid the splash daemon involved in a recorded failed boot; retain a text unlock prompt |

There is **no AMD blacklist in the final configuration**. Low/high performance policies are workarounds for the automatic policy; they are not equivalent to switching the GPU off. Low mode limits performance. High mode increases heat and power draw.

T2 Linux documents both selecting Intel as primary and forcing AMD performance levels for this class of instability. Its alternative `amdgpu.dpm=0` suggestion did **not** work on this particular kernel/GPU combination. [T2 hybrid graphics guide](https://wiki.t2linux.org/guides/hybrid-graphics/), [T2 device support](https://wiki.t2linux.org/state/), [AMD kernel parameters](https://docs.kernel.org/gpu/amdgpu/module-parameters.html).

## Getting into an otherwise unbootable installation

At the Limine menu, select the installed T2 kernel, press **E**, and append these parameters to its existing command line:

```text
apple_gmux.force_igd=1 module_blacklist=amdgpu
```

Press **F10** to boot. A USB keyboard may be needed because Touch Bar function keys may not be available. This is a temporary Intel-only recovery boot. Keep all existing disk-unlock and root parameters.

If the desktop prevents troubleshooting, also append:

```text
systemd.unit=multi-user.target plymouth.enable=0
```

The AMD blacklist must have **no trailing period**. `module_blacklist=amdgpu.` did not block the real module in the observed session. A boot-menu edit lasts for that boot only.

## Apply the configuration

These instructions target the tested Omarchy/Limine UKI layout. First confirm the model and tools:

```bash
cat /sys/class/dmi/id/product_name
uname -r
lspci -nnk | grep -A3 -E 'VGA|Display'
command -v limine-mkinitcpio limine-entry-tool
```

Inspect the five files under [`config/etc`](config/etc) before installing them. They are the final configuration files used in the repair, without machine-specific disk identifiers.

### 1. Preserve a working recovery boot and existing configuration

Do this while the normal UKI is a known-working Intel-only image. The following backup directory and recovery filename must be new; choose another name if either already exists.

```bash
(
  set -eu
  backup=/root/t2-graphics-before
  recovery=/boot/EFI/OmarchyTests/intel-recovery.efi
  sudo test ! -e "$backup"
  sudo test ! -e "$recovery"
  sudo install -d -m 700 "$backup" /boot/EFI/OmarchyTests
  sudo cp -a /etc/modprobe.d /etc/mkinitcpio.conf.d \
    /etc/limine-entry-tool.d /etc/udev/rules.d "$backup/"
  sudo cp -a /boot/limine.conf "$backup/"
  sudo cp -a /boot/EFI/Linux/omarchy_linux-t2.efi "$backup/"
  sudo cp -a /boot/EFI/Linux/omarchy_linux-t2.efi "$recovery"
  sudo limine-entry-tool --add-efi 'Omarchy Intel recovery' \
    "$recovery" --priority 30 \
    --comment 'Working Intel-only boot; AMD disabled'
)
```

Adjust the source UKI path if your installation uses a different name. A static recovery image contains its original embedded command line and kernel. It is **not automatically refreshed after kernel upgrades**; an old kernel may no longer match the modules on the current root filesystem. This recovery entry was verified with the same installed kernel used for the repair.

### 2. Install the five files

Run from the root of this repository after making the backup:

```bash
(
  set -eu
  for path in \
    etc/modprobe.d/apple-gmux.conf \
    etc/mkinitcpio.conf.d/zz-local-t2-graphics.conf \
    etc/mkinitcpio.conf.d/zz-local-t2-amd-power.conf \
    etc/udev/rules.d/99-local-t2-amd.rules \
    etc/limine-entry-tool.d/zz-local-t2-graphics.conf
  do
    sudo install -D -m 644 "config/$path" "/$path"
  done
)
```

Inspect your existing configuration for any other AMD blacklist or conflicting options:

```bash
sudo grep -RnsE 'blacklist.*amdgpu|amdgpu\.(dpm|runpm)|force_igd' \
  /etc/modprobe.d /etc/limine-entry-tool.d /etc/default/limine
```

Remove only earlier workarounds you recognize and have backed up. In particular, a remaining `module_blacklist=amdgpu` would prevent the final configuration from enabling the GPU.

### 3. Rebuild through Limine

```bash
sudo limine-mkinitcpio linux-t2
```

The tested installation had **no `/etc/mkinitcpio.d/*.preset` files**. `mkinitcpio -P` is therefore not the correct rebuild procedure for this layout. `limine-mkinitcpio linux-t2` generates the UKI and updates the normal Limine entry and its checksum.

Before rebooting, inspect the generated command line and menu:

```bash
sudo limine-entry-tool --get-cmdline linux-t2
sudo limine-entry-tool --tree
```

The normal command line should contain the settings in this repository, preserve your existing disk parameters, and contain neither an AMD blacklist nor `systemd.unit=multi-user.target`. Reboot into the regular **Omarchy → linux-t2** entry and unlock the disk. A text password prompt is intentional.

## Verify AMD is actually usable

```bash
cat /proc/cmdline
lspci -nnk | grep -A3 -E 'VGA|Display'
cat /sys/module/amdgpu/parameters/dpm
cat /sys/module/amdgpu/parameters/runpm
cat /sys/bus/pci/devices/0000:03:00.0/power_dpm_force_performance_level
hyprctl monitors
hyprctl configerrors
```

Expected: `amdgpu` is the AMD device's **driver in use**, DPM is `1`, runtime PM is `0`, performance level is `low`, and only the real internal screen is active when no external display is connected. Merely seeing `Kernel modules: amdgpu` is not proof that the driver bound successfully.

For graphics diagnostics, install `mesa-utils`, `vulkan-tools`, and optionally `glmark2` through your normal package-management workflow. From a terminal in the desktop:

```bash
DRI_PRIME=pci-0000_03_00_0 glxinfo -B
DRI_PRIME='pci-0000_03_00_0!' vulkaninfo --summary
DRI_PRIME=pci-0000_03_00_0 glmark2 --off-screen --size 800x600 \
  -b build:duration=10 -b shading:duration=10
```

Look for an AMD renderer and `Accelerated: yes`, not a software renderer. The trailing `!` in the Vulkan selection restricts enumeration to the selected GPU. The PCI address is specific to the tested machine; verify it before reusing these commands elsewhere.

## Use AMD for an application

Intel remains responsible for the desktop's internal-panel output. Launch a demanding application with AMD rendering:

```bash
DRI_PRIME=pci-0000_03_00_0 your-application
```

For applications that need more performance, the tested high policy is:

```bash
echo high | sudo tee /sys/bus/pci/devices/0000:03:00.0/power_dpm_force_performance_level
```

Return to the default low policy afterward:

```bash
echo low | sudo tee /sys/bus/pci/devices/0000:03:00.0/power_dpm_force_performance_level
```

These commands change the GPU-wide policy until changed again or rebooted. They do not implement automatic per-application switching. High mode passed a short test; it was not certified for long gaming sessions. Do not assume `auto` is stable just because low and high passed.

## Limitations and recovery

- One machine was tested. The original power-off's exact kernel/firmware cause was not captured.
- The fix was tested as a combination; individual changes were not fully isolated.
- AMD remains available, but the default low policy limits its throughput. Runtime power management is disabled, which can increase idle power use.
- Some startup EDID errors for the unused `eDP-2` connector remain in the kernel log. It is excluded from the active desktop.
- Suspend/resume, hibernation, external monitors, long-duration load, and future kernel upgrades have not been validated.
- No live Hyprland monitor-disable override is installed. That approach caused a separate compositor crash during diagnosis.

See [recovery and rollback](docs/recovery.md) for restoring Intel-only operation. The `/etc` drop-ins are outside Omarchy's package-owned directory; future rebuilds are expected to consume them, but the behavior must still be checked after upgrades.

## Sources

- [T2 Linux hybrid graphics guide](https://wiki.t2linux.org/guides/hybrid-graphics/): Intel display routing and AMD performance-level workarounds.
- [T2 Linux device support](https://wiki.t2linux.org/state/): AMD instability and limitations.
- [Linux AMD module parameters](https://docs.kernel.org/gpu/amdgpu/module-parameters.html): DPM and runtime PM settings.
- [Linux display-mode parameters](https://docs.kernel.org/fb/modedb.html): disabling individual outputs at boot.
- [Omarchy discussion #7855](https://github.com/omacom/omarchy/discussions/7855): related black-screen reports on the 2019 16-inch model. Its live monitor and rebuild suggestions were not sufficient for this installed stack.

The configuration, observations, and validation records here came from direct inspection and testing of the affected laptop. Raw logs, core dumps, credentials, disk identifiers, and webcam images are not included.
