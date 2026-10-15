# HP OMEN 16-ap0xxx (board 8E35): Linux shutdown leaves the NVIDIA dGPU powered

Research notes on existing fixes, compiled 2026-10-15.

## This machine

| Field | Value |
|---|---|
| Model | OMEN Gaming Laptop 16-ap0xxx (SKU BP1Q1UA#ABA) |
| Board | 8E35 |
| BIOS | AMI F.13 (2026-05-12) |
| CPU / iGPU | Ryzen 9 8940HX / Radeon 610M |
| dGPU | RTX 5060 Laptop (PCI 10DE:2D19, subsys 8E35103C) |
| DSDT | `HPQOEM` / `8E35`, OEM revision `0x01072009`, 110218 bytes, SHA-256 `50c13dce0bb725b6b3ce3f797dce8e3680ecc76c42d6f1874135bc53bd1b3d21` (read from the Windows registry) |

The DSDT contains `_PTS`, `WQBZ`, `WQBE`, `BF01`, `WBU1`, `WBU2` and `NVDE`. It does not contain `OMPR` or `PG00`, which live in an SSDT. That matches the layout described by omen-acpi. Its patched DSDT variants use OEM revisions `0x0107200A`/`B`/`C`, which implies a stock revision of `0x01072009`, the same as this machine's.

## Symptom

After `poweroff`, the screen goes dark and the kernel prints `reboot: Power down`. The dGPU stays powered anyway, and the machine draws about 19–21 W in a fake S5 until the battery is empty. The area under the GPU stays warm. Shutting down from Windows works normally. Measured baselines are 20.05, 19.73, 20.33 and about 21 W, on several units.

## Root cause: two causes together, plus one false lead

1. **Kernel side.** `pci_device_shutdown()` in `drivers/pci/pci-driver.c` calls `pm_runtime_resume(dev)` on every PCI device, and it has done so since 2012 (commit `3ff2de9ba1a2`). This wakes the dGPU from runtime D3cold before `nvidia`'s `.shutdown` runs, so the GPU is in D0 when the system enters S5.
2. **Firmware side.** If the dGPU is in D0 when the system enters S5, the board never cuts its power rail. `_PTS(5)` has no dGPU power-down step. The power-down sequence does exist in the firmware, but only on the runtime-PM path:
   ```
   PEGP._PS3:  If (OMPR == 0x03) { PG00._OFF() ... }
   PG00._OFF:  If (NVDE != One) { Return (Zero) }   // early-out
   ```
   `NVDE` is set to 1 by the NVIDIA driver through the Optimus `_DSM` calls (NBCI, GPS, NVJT, NVOP), and `_WAK` clears it. With nouveau, or with no driver bound (VFIO), `PG00._OFF` does nothing.
3. **False lead: `WQBZ` / `AE_AML_BUFFER_LIMIT`.** The forum threads blame `\_SB.WMID.WQBZ` reading past the end of the 0x32-byte `BF01` buffer. That bug is real and makes boot logs noisy, but it is a separate problem. The S5-only patches fix the drain without touching `WQBZ`.

Evidence that both causes are needed: when the firmware powers off from GRUB `halt`, with no Linux involved, the draw is about 1.05 W. When the kernel is prevented from resuming the dGPU, the draw is about 0.45 W. When the firmware runs the dGPU power-off at S5, the draw is about 1–2 W.

## Things that do NOT work (tested by others)

- Kernel parameters: `acpi_osi="Windows 20xx"`, `acpi_sleep=s5`, `acpi_rev_override=5`, `acpi=force`, `reboot=pci`, `reboot=efi`
- Kernels 6.12.77 through 7.0-rc6, on Fedora, Arch, CachyOS and Ubuntu
- Disabling USB charging, wake-on-LAN and lid wake in the BIOS
- BIOS updates up to F.13. There is no HP fix and no acknowledgement from HP, and HP support told one user to use Windows.
- NVIDIA driver versions 580.x, 595.x and 610.x. No NVIDIA staff have responded on their forum.

## Existing solutions

| Project | Mechanism | Secure Boot | Handles dGPU awake at shutdown? | Validated on | Measured |
|---|---|---|---|---|---|
| [Adiakys/omen16-s5-fix](https://github.com/Adiakys/omen16-s5-fix) | DKMS module. Loads a small SSDT (`\OS5F`: `OMPR = 3; PEGP._PS3()`) with `acpi_load_table()` and calls it from a `SYS_OFF_MODE_POWER_OFF_PREPARE` handler at `SYS_OFF_PRIO_HIGH`, which runs right after ACPI's `_PTS(5)` handler. Checks DMI for board 8E35. | Yes (signed module + MOK) | Yes | 16-ap0004, RTX 5070, F.11, Ubuntu 26.04, kernel 7.0 | 21 W → 1–2 W |
| [paolo-de-marinis/omen-acpi](https://github.com/paolo-de-marinis/omen-acpi) ([mirror/fork](https://github.com/WoofahRayetCode/omen-acpi)) | DSDT override through an uncompressed early cpio in the initrd. Patches `_PTS(5)` with the same `OMPR`/`_PS3` sequence. Variants: `s5`, `combined` (adds bounds checks to the `WQBZ` loops), `s5-vfio` (also sets `NVDE = 1`). | No (lockdown ignores table overrides) | Yes | OMEN MAX 16-ap0006sl, RTX 5070, **F.13**, CachyOS + Limine. Community reports: GRUB on Ubuntu ([#3](https://github.com/paolo-de-marinis/omen-acpi/issues/3)), Pop!_OS + kernelstub ([#2](https://github.com/paolo-de-marinis/omen-acpi/issues/2)) | ~21 W → ~1 W; subjective "cool to the touch" |
| [December172/s5-shield](https://github.com/December172/s5-shield) | DKMS module. A reboot notifier runs before `device_shutdown()`: it waits up to 20 s for the dGPU to settle into D3cold, calls `__pm_runtime_disable()` so the PCI core's resume fails, and sets `.shutdown` to NULL for `nvidia` and `snd_hda_intel`. Optional fallback powers off from systemd-boot. | Signed module | **No.** With a held reference (external display, CUDA, PRIME) the draw is 18.5 W | 16-ap0xxx, **RTX 5060, F.13**, 8945HX, Arch, kernel 7.2.8, nvidia-open 615 | 20.33 W → 0.42–0.43 W (overnight, battery-ledger method) |
| [AnxoCalvo/s5-poweroff-fix](https://github.com/AnxoCalvo/s5-poweroff-fix) | Same idea as s5-shield, packaged as a Fedora akmod (`s5_pmrt_arm`) with systemd hooks. If the dGPU is still in D0 after 90 s, it reboots into GRUB `halt`. | akmod | Via the GRUB-halt fallback | 16-ap0xxx, Fedora 44 (plus a second unit on Arch) | ~20 W → 0.45–0.47 W over 4 nights |
| [iammhador/hp-omen-linux-shutdown-fix](https://github.com/iammhador/hp-omen-linux-shutdown-fix) | Reboots into Windows through rEFInd plus a flag file, and a Windows scheduled task runs `shutdown /s`. | n/a | Yes | OMEN 16, Fedora | n/a. Fragile: Fast Startup must be off, and the ESP can be corrupted. |

Caveats that apply to all of them: every project is from Aug–Oct 2026 and was validated on one or two machines. omen-acpi and s5-shield both say they were written with AI assistance. Both module approaches change live kernel or driver state at poweroff. Read the code before installing any of them.

## Upstream status

- **Kernel:** Mario Limonciello sent "PCI: Avoid runtime resuming devices if system is shutting down" in Dec 2023. It skipped `pm_runtime_resume()` when `system_state == SYSTEM_POWER_OFF`. Rafael Wysocki objected: "What's the guarantee that the driver's shutdown callback will not access MMIO regions of a runtime-suspended device?" The patch was never merged. A related 2024 thread, "PCI/PM: Put devices to low power state on shutdown", also exists. Its outcome has not been checked yet.
- **HP:** no BIOS fix up to F.13, and no public acknowledgement.
- **NVIDIA:** no response on the developer-forum thread.

## Trade-offs between the two working approaches

- **Firmware-sequence approach** (omen16-s5-fix, omen-acpi): powers the dGPU off no matter what state it is in, so it is robust to external monitors and CUDA. It depends on `NVDE == 1`, which means the NVIDIA proprietary or open driver must be loaded. The module variant works under Secure Boot; the DSDT override does not.
- **Prevent-resume approach** (s5-shield, s5-poweroff-fix): has the lowest measured draw (~0.4 W). It only works when runtime D3cold is functioning and the dGPU is idle at shutdown. It also sets `.shutdown` pointers to NULL on live drivers, which is invasive.

## Decision for this machine

- **Distro:** Omarchy. It is Arch-based, uses Limine with UKIs, and installs `nvidia-open` automatically on Turing and newer GPUs. CachyOS is not required: the chosen fix is a kernel module and does not depend on the bootloader. CachyOS is only a fallback, because omen-acpi was validated on CachyOS + Limine + F.13.
- **Driver:** `nvidia-open`. It is required for the RTX 50 series anyway, and the fix depends on it because the NVIDIA driver is what sets `NVDE = 1`. With nouveau, `PG00._OFF` would do nothing.
- **External monitors:** one monitor on HDMI, or two on USB-C. If either port is wired to the dGPU (on similar OMEN AMD models, HDMI goes to the dGPU), the dGPU will be awake at shutdown in that setup. That rules out the prevent-resume approach (s5-shield, s5-poweroff-fix).
- **Chosen fix:** [Adiakys/omen16-s5-fix](https://github.com/Adiakys/omen16-s5-fix), the firmware-sequence approach as a module. Omarchy requires Secure Boot off, so no MOK enrollment or module signing is needed.
- **Booting:** with the firmware F9 menu, not a dual-boot menu. Omarchy's Limine still boots Linux itself.

## Code review: omen16-s5-fix (2026-10-15)

- `src/omen_s5.c` is about 60 lines. Each step is a failure exit; on success it registers the handler:
  - The DMI check (`HP` / `8E35`) must match, or the module exits.
  - `\_SB.PCI0.GPP0.PEGP._PS3` and `.OMPR` must exist, or the module exits.
  - If `\OS5F` is not already defined, it loads the embedded SSDT with `acpi_load_table()`. If that fails, the module exits.
  - It registers `SYS_OFF_MODE_POWER_OFF_PREPARE` at `SYS_OFF_PRIO_HIGH`. ACPI's own `_PTS(5)` handler is registered at `SYS_OFF_PRIO_FIRMWARE`, which runs first, so `\OS5F` runs after `_PTS(5)` and after `device_shutdown()`. The handler only fires on poweroff, never on reboot.
- The embedded AML (`src/os5f_aml.h`) was decoded by hand. It matches `src/os5f.asl` byte for byte:
  - `External`s wrapped in `If (Zero)`, as iasl emits them.
  - `Method (\OS5F, 0, Serialized)` with two `CondRefOf` guards.
  - `Store (3, OMPR)`, then `PEGP._PS3()`, then `Return (One)`, and otherwise `Return (Zero)`.
  - The length is 211 bytes, matching the header, and the checksum is valid (the byte sum mod 256 is 0).
- There is no `module_exit`, so once loaded the module stays active until reboot. This is acceptable because the loaded SSDT cannot be cleanly unloaded anyway.
- License inconsistency: the source says `GPL-2.0`, but the repo's `LICENSE` is GPL-3.0.
- `install.sh` supports pacman (`dkms linux-headers`). It writes `/etc/modules-load.d/omen-s5.conf` and an XDG autostart entry that checks the module loaded. Whether that autostart entry fires under Hyprland/uwsm has not been checked.

## Pre-install checklist (in Windows)

1. Check Settings → Privacy & security → Device encryption. Omarchy's dual-boot install requires BitLocker to be off; the non-admin check from this session was inconclusive. Save the recovery key before changing any BIOS security settings.
2. Shrink `C:` in Disk Management to leave unallocated space. The disk is a single 954 GB NVMe, and `C:` currently takes 949 GB of it.
3. Note which GPU drives each external port: plug in a monitor, then check Settings → System → Display → Advanced display.
4. Turn off Secure Boot in the HP BIOS (F10).
5. Run the Omarchy ISO's free-space install, then boot it from the F9 menu.

## Post-install steps (in Omarchy)

1. Confirm `nvidia` is loaded and runtime PM is working: `lsmod | grep nvidia`, and `/sys/bus/pci/devices/0000:01:00.0/power/runtime_status`.
2. Record a baseline drain: charger out, read `energy_now`, power off for 30+ minutes, then boot and read it again.
3. Dump the ACPI tables (`acpica`: `acpidump -b`, `iasl -d`). Confirm that the `PEGP._PS3` / `PG00._OFF` / `NVDE` logic on this BIOS matches the snippet above.
4. Install omen16-s5-fix, then check `dmesg | grep omen_s5` for `ready`.
5. Measure again in three setups: no monitor, HDMI monitor attached at shutdown, and USB-C monitors attached at shutdown.

## Open questions to settle on this machine (from Linux)

1. Which GPU are the HDMI and USB-C ports wired to?
2. Does `sha256sum /sys/firmware/acpi/tables/DSDT` match the registry copy above? It does not match the hash `9f0b43f2…` quoted in omen-acpi issue #3, but it is unclear what that hash covers.
3. Which SSDT holds `PEGP._PS3` / `PG00`, and does its logic match the snippet above? (`acpidump`, `acpixtract`, `iasl -d`)
4. Does runtime D3cold work? Check `/sys/bus/pci/devices/0000:01:00.0/power/runtime_status` and `power_state` while idle. Older forum posts reported that writing `power/control` hung the kernel.
5. Baseline the drain with a battery-ledger measurement before and after any fix: charger unplugged, read `energy_now` before poweroff and after 30+ minutes off.

## Sources

- Arch forums: https://bbs.archlinux.org/viewtopic.php?id=313030
- Fedora Discussion: https://discussion.fedoraproject.org/t/technical-issue-incomplete-shutdown-on-hp-omen-16-ap0038ns/184041
- CachyOS forum: https://discuss.cachyos.org/t/technical-issue-incomplete-shutdown-on-hp-omen-16-ap0038ns/26236
- HP Support Community: https://h30434.www3.hp.com/t5/Gaming-Notebooks/After-shuting-down-from-a-Linux-distribution-HP-OMEN-16/td-p/9624996
- NVIDIA dev forum: https://forums.developer.nvidia.com/t/hp-omen-16-rtx-5060-battery-drains-after-shutdown-on-linux-gps-acpi-dsm-bug/364074
- LKML, Limonciello patch review: https://lkml.iu.edu/hypermail/linux/kernel/2312.1/07216.html and https://lkml.iu.edu/2312.1/07234.html
- Repos: listed in the table above
