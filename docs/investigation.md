# Investigation record

## Original failure and initial recovery

The reported symptom was a black screen after the first boot's password prompt, followed by a brief fan surge and power-off. A later attempt displayed a gray screen and a small dark rectangle before powering off.

The initial report named Omarchy 4.0.2 and a 15-inch 2019 MacBook. By the time SSH access was available, inspection showed Omarchy 4.0.3, a T2 7.2.4 kernel, and **MacBookPro16,1**, the 16-inch 2019 model. This record does not establish whether an Omarchy release change contributed to the failure.

T2 documentation mentions CPU CATERR for similar symptoms, but no captured log proved CATERR on this machine. The correct conclusion is a graphics-related boot workaround supported by testing, not a confirmed hardware fault or a fully isolated kernel defect.

The first remotely inspected command line contained `module_blacklist=amdgpu.`. The trailing period made this different from the actual module name, and `amdgpu` was bound to the Radeon device. Meanwhile, the existing gmux file was correct: the kernel reported `Switching to IGD`, and vgaswitcheroo showed Intel as the active display adapter.

That combination produced an Intel `eDP-1` display and a bogus AMD `eDP-2` display with a 0×0 mode. AMD logged missing EDID, repeated stream-allocation errors, and fence fallback messages. A preceding failed boot recorded a Plymouth SIGSEGV and no usable saved core for that process. The evidence does not prove the Plymouth crash alone caused the subsequent power-off.

Correctly blocking AMD and loading gmux/i915 early produced a successful Intel-only boot. This was retained as the recovery configuration; it did not count as repairing AMD.

## The phantom-output detour

With AMD blocked, its original firmware framebuffer remained as `Unknown-1`, alongside the real Intel panel. The mapping under `/sys/class/drm` identified its driver as `simple-framebuffer`.

Attempting to disable this output live with a Hyprland Lua monitor rule triggered an assertion in `Render::GL::CGLFramebuffer::internalAlloc`, reached through monitor framebuffer allocation and rendering. Hyprland aborted and its launcher restarted it with `--safe-mode`.

The monitor file was restored byte-for-byte from its backup. Restarting SDDM returned Hyprland to normal mode. **The final fix contains no Hyprland monitor override.** It excludes the phantom connectors through kernel parameters before the compositor starts instead.

## AMD test 1: disable DPM entirely

A separate UKI and Limine entry were created so the working Intel boot remained available. The test used:

```text
apple_gmux.force_igd=1 amdgpu.dpm=0 video=eDP-2:d video=Unknown-1:d plymouth.enable=0 systemd.unit=multi-user.target
```

The machine reached a text login and SSH, but AMD failed initialization. Relevant messages included:

```text
amdgpu: smu firmware loading failed
amdgpu_device_ip_init failed
Fatal error during GPU init
probe with driver amdgpu failed with error -95
```

The module was loaded, but the PCI device had no driver bound and no AMD render node. This distinction matters: booting without a shutdown was not evidence of a working AMD GPU.

A subsequent module reload with DPM enabled could not reattach the device; explicitly binding it returned EAGAIN. A clean boot was used for the next test rather than continuing to manipulate the failed device state.

## AMD test 2: enabled DPM with a forced performance level

The second clean boot used `amdgpu.dpm=1 amdgpu.runpm=0` and a udev rule selecting `low` for the AMD performance level. The rule was explicitly included in the initramfs so it applied during early initialization. Intel routing and the boot-time phantom-output exclusions were retained.

The SMU initialized, `amdgpu` bound to the Radeon, and an AMD render node appeared. Hardware EGL/OpenGL and Vulkan enumeration passed. Starting SDDM then produced a normal Hyprland desktop with only the real panel active.

Explicit AMD rendering benchmarks completed at both `low` and `high` performance levels. Low was restored after testing. The resulting combination was selected for permanent configuration.

## Why the rebuild command changed

Inspection found an empty `/etc/mkinitcpio.d` directory and a Limine UKI workflow. The installed `limine-mkinitcpio` script invokes the package-aware UKI builder, obtains its kernel command line from the Limine configuration, installs the generated image, and updates the boot menu.

For this layout the final rebuild used:

```bash
sudo limine-mkinitcpio linux-t2
```

The generated UKI's embedded command line, initramfs contents, and Limine image checksum were inspected. Merely editing a module option and invoking a nonexistent preset workflow would not establish that the firmware was loading a changed image.

## Interpretation

The observations establish that the GPU can initialize and render on this machine with the documented combination. They do not establish which one of the clock policy, runtime PM, early gmux loading, phantom-output exclusion, or splash avoidance is individually sufficient to prevent every original failure.

A full root-cause investigation would require controlled testing of those variables, longer workloads, and possibly kernel instrumentation. The working configuration is retained rather than presenting an untested minimal fix as proven.
