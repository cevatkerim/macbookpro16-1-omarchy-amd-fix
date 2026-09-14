# Validation

Tests were run on 2026-09-14 on the hardware and package versions listed in the README.

## Working diagnostic boot

| Check | Observed result |
| --- | --- |
| AMD PCI driver binding | `Kernel driver in use: amdgpu` on `0000:03:00.0` |
| Intel PCI driver binding | `i915` on `0000:00:02.0` |
| AMD power-management support | `dpm=1`, `runpm=0` |
| Startup performance policy | `low` |
| AMD render device | Present, through the PCI-specific render symlink |
| EGL/OpenGL | Hardware AMD renderer; OpenGL 4.6 and OpenGL ES 3.2 |
| GLX | `direct rendering: Yes`, `Accelerated: yes`, 4096 MB video memory |
| Vulkan | `AMD Radeon Graphics (RADV NAVI14)`, API 1.4.354 |
| Active desktop output | Only `eDP-1`, 3072×1920 at 60 Hz |
| Hyprland configuration errors | None |
| GPU reset or power-off during rendering tests | None observed |

Representative renderer identifiers:

```text
AMD Radeon Graphics (radeonsi, navi14, ACO, DRM 3.64, 7.2.4-arch1-Watanare-T2-2-t2)
AMD Radeon Graphics (RADV NAVI14)
```

## Short rendering tests

The same two off-screen glmark2 scenes ran on the explicitly selected Radeon GPU at 800×600, for 10 seconds per scene.

| Performance policy | Build scene FPS | Shading scene FPS | Two-scene score |
| --- | ---: | ---: | ---: |
| `low` | 3508 | 2865 | 3185 |
| `high` | 15597 | 13865 | 14730 |

These are brief synthetic tests, not the full glmark2 suite, a gaming benchmark, or an endurance test. Their main purpose was to verify that AMD performed real accelerated rendering in both policies without a reset or shutdown.

After the high test, the sampled AMD sensor readings were 73°C edge, 75°C junction, and approximately 16 W. The subsequent state was returned to `low`. These readings are snapshots, not thermal limits or a complete temperature trace.

## Permanent boot image

Before the final reboot, verification confirmed:

- The regular UKI embeds the AMD-enabled parameters and has no AMD blacklist.
- The normal entry has no `systemd.unit=multi-user.target`, allowing the desktop to start normally.
- The initramfs contains gmux, i915, amdgpu, the gmux option, and the early AMD udev rule.
- The Limine entry's image checksum matches the generated UKI.
- The separately preserved Intel recovery UKI still embeds the working AMD blacklist.

The final regular **Omarchy → linux-t2** boot was then verified over SSH:

- The machine reached its normal desktop automatically after disk unlock.
- The active command line enabled AMD and contained no blacklist or text-mode target override.
- AMD was bound to its PCI device and the startup performance level was `low`.
- `glxinfo -B` explicitly targeting AMD again reported `direct rendering: Yes`, `Accelerated: yes`, and 4096 MB of video memory.
- Hyprland was running in normal mode, with no configuration errors and only `eDP-1` active.
- SDDM, SSH, and the T2 fan daemon were active.
- No segfault, core dump, GPU reset, ring timeout, or CATERR message matched the final boot's journal check.

The desktop had been running for just over three minutes at this check. This confirms the permanent configuration survived reboot; it is not a long-duration stability claim.

## Not established

- The precise cause of the original firmware/kernel-level shutdown.
- Stability of AMD's automatic performance policy.
- Sleep, hibernation, external-display behavior, prolonged gaming/rendering, or future software versions.
- That every setting in the working combination is individually necessary.

Kernel messages about the unused AMD eDP connector can still occur during detection. No claim is made that the entire kernel log is error-free.
