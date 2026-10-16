# omen16-linux-fix

Makes **HP OMEN 16-ap0xxx (board 8E35)** laptops actually turn off when Linux shuts down.

Without the fix, the screen goes dark but the NVIDIA GPU stays powered. The laptop then draws **about 20 W while "off"**, gets warm, and drains a full battery in a few hours. Windows doesn't have this problem. HP has not fixed it in any BIOS up to F.13.

This repo wraps the community fix [Adiakys/omen16-s5-fix](https://github.com/Adiakys/omen16-s5-fix) with:
- an installer for Arch-based systems (Omarchy, CachyOS)
- a firmware check that confirms your BIOS matches what the fix expects
- a battery ledger that measures how much power the laptop really uses while off

Background, other approaches, and sources are in [RESEARCH.md](RESEARCH.md).

> **Status:** works on this repo's laptop (16-ap0xxx, Ryzen 9 8940HX, RTX 5060, BIOS F.13, kernel 7.2.5, nvidia-open 610.57.04). Measured with the fix, on battery: **0.31 W** while off overnight (2.05 Wh over 6.7 h, with the dGPU awake in D0 at shutdown), and 1.53 W over a 20-minute window, where boot overhead dominates. Both windows used only the built-in display; runs with external monitors attached are still to come. The same mechanism was measured at 21 W → 1–2 W on a 16-ap0004 (BIOS F.11).

---

## How it works

### What a normal shutdown looks like

`systemctl poweroff` ends in the kernel's `kernel_power_off()` (`kernel/reboot.c`). It runs these steps in order:

```
kernel_power_off()
 ├─ device_shutdown()                     every device's .shutdown()
 │    └─ pci_device_shutdown(dGPU)
 │         ├─ pm_runtime_resume()         wakes the dGPU out of D3cold  ◄── problem 1
 │         └─ nvidia .shutdown()
 │
 ├─ power-off-prepare handlers            (highest priority first)
 │    └─ [224] acpi_power_off_prepare()   runs the firmware's _PTS(5)   ◄── problem 2
 │
 ├─ syscore_shutdown()
 ├─ "reboot: Power down"
 └─ acpi_power_off()                      enter S5: the board turns off
```

**Problem 1 (Linux).** `pci_device_shutdown()` always resumes the device before shutting it down. This has been true since 2012. So a dGPU that had already powered itself off (D3cold) is switched fully back on just before the system powers off. A kernel patch to skip this was proposed in 2023 and never merged.

**Problem 2 (HP firmware).** If the dGPU is on when the board enters S5, the board does not cut its power rail. HP's S5 preparation method `_PTS(5)` has no step that turns the dGPU off. The firmware *does* contain the power-off code, but it is only used for runtime power saving, and it sits behind two guards:

```
PEGP._PS3   (dGPU "go to D3")      If (OMPR == 0x03) { PG00._OFF() }   // OMPR: "power-off armed" flag
PG00._OFF   (cut the power rail)   If (NVDE != One)  { Return }        // NVDE: set by the NVIDIA driver
```

With both problems present, the board reaches a fake S5 with the GPU still powered. You may see `AE_AML_BUFFER_LIMIT` / `WQBZ` errors in `dmesg`. They come from a separate, harmless HP firmware bug and are **not** the cause.

### What the fix does

The `omen_s5` kernel module adds one step at the end of the shutdown sequence:

```
 ├─ power-off-prepare handlers
 │    ├─ [224] acpi_power_off_prepare()   _PTS(5)
 │    └─ [192] omen_s5                    \OS5F:  OMPR = 3
 │                                                PEGP._PS3()  →  PG00._OFF()  →  dGPU power rail off
```

When the module loads, it does three things:
1. It checks that this is an HP board `8E35` and that `\_SB.PCI0.GPP0.PEGP._PS3` and `.OMPR` exist in the firmware.
2. It adds a tiny ACPI table (SSDT) to the running system with one method, `\OS5F`:
   ```
   Method (\OS5F, 0, Serialized) {
       \_SB.PCI0.GPP0.PEGP.OMPR = 0x03     // arm the firmware's own power-off path
       \_SB.PCI0.GPP0.PEGP._PS3 ()         // ...and take it
   }
   ```
3. It registers a power-off-prepare handler at priority 192. ACPI's handler runs at priority 224, so `\OS5F` runs immediately after `_PTS(5)`. By then every driver has already shut down, so cutting the GPU's power cannot disturb anything.

Some properties of this design:
- **It uses HP's own power-off code.** The fix only arms the flag and calls the method; it doesn't drive GPIOs directly.
- **It doesn't matter what the GPU is doing.** External monitors on the NVIDIA outputs, CUDA, or a busy GPU at shutdown are all fine. The fix doesn't try to keep the GPU asleep; it turns it off at the very end.
- **It only affects poweroff.** The handler isn't registered for reboot.
- **It touches nothing permanent.** No BIOS flashing, no edited firmware tables on disk, and no bootloader changes. Uninstall it and the next boot is stock.
- **It needs the NVIDIA driver loaded** (`nvidia` / `nvidia-open`), because `PG00._OFF` checks `NVDE == 1`, which the NVIDIA driver sets. With nouveau or no driver, the call runs but does nothing.
- **It can't be unloaded at runtime.** The module has no exit function, so `rmmod` won't work. Uninstall it and reboot.

---

## What's in this repo

| Path | What it does | Needs root |
|---|---|---|
| `bin/omen-status` | Read-only health check: board/BIOS, Secure Boot, NVIDIA driver and power state, which GPU drives each display, whether the fix is loaded | no |
| `bin/omen-drain` | Battery ledger. A systemd service logs battery energy at every shutdown and boot. `report` shows the watts drawn while off. | `install`/`uninstall` only |
| `bin/omen-acpi-check` | Dumps and disassembles the ACPI tables, then checks `_PTS`, `PEGP._PS3` (OMPR guard), `PG00._OFF` (NVDE guard), and the DSDT hash | yes |
| `bin/omen-s5-install` | Installs `omen_s5` via DKMS from a pinned upstream commit, after checking file hashes. Supports `uninstall`. | yes |
| `RESEARCH.md` | Root-cause research, every known fix compared, sources | |

---

## Setup

### Before installing Linux (in Windows)

1. Turn off **BitLocker / Device encryption** (Settings → Privacy & security → Device encryption) and save the recovery key. Omarchy's same-drive install requires it to be off.
2. Turn off **Fast Startup**: Control Panel → Power Options → *Choose what the power buttons do* → *Change settings that are currently unavailable* → untick *Turn on fast startup*. With it on, Windows "shutdown" is really a partial hibernate. That leaves the Windows partition unsafe to mount from Linux and breaks clean switching between OSes.
3. Make Windows keep the **hardware clock in UTC**, as Linux does. Otherwise Linux's clock is 7–8 h off after every Windows session until it syncs online. Run this in an **admin** terminal:
   ```
   reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_QWORD /d 1 /f
   ```
4. Shrink `C:` in Disk Management to leave unallocated space for Linux.
5. Optional: plug in your monitors and check Settings → System → Display → Advanced display to see which GPU drives HDMI and USB-C.
6. In the BIOS (F10), turn off **Secure Boot**. Omarchy requires this, and it also means the module doesn't need signing.
7. Install Omarchy with the **free-space** option. See [Switching between Windows and Omarchy](#switching-between-windows-and-omarchy).

Use NVIDIA's driver (`nvidia-open`), not nouveau. Omarchy installs `nvidia-open` automatically for RTX cards.

### Switching between Windows and Omarchy

Pick the OS at power-on with the BIOS boot menu (**F9**). There is no shared dual-boot menu:

- **Each OS boots independently.** Omarchy's free-space install creates its own ~2 GB EFI partition (`omarchy-efi`) with its own boot loader (Limine), and leaves Windows' EFI partition untouched. Limine boots only Omarchy; don't add Windows to it (`limine-scan`).
- **The default OS** (the one that starts when you don't press F9) is set by the BIOS (F10) boot order. Windows updates sometimes move *Windows Boot Manager* back to the top.

**If Omarchy doesn't appear in F9** and the laptop boots straight into Windows: HP firmware is known to hide or drop Linux boot entries. Omarchy's free-space installer also doesn't currently install the backup boot file that firmware falls back to (`EFI/BOOT/BOOTX64.EFI`; see [omarchy#11997](https://github.com/omacom/omarchy/issues/11997) and [omarchy-iso#158](https://github.com/omacom/omarchy-iso/pull/158)). To recover:

1. Press **F9** → **Boot From EFI File**. Pick the Omarchy EFI partition, then `EFI` → `Limine` → `limine_x64.efi` (or whichever `.efi` file is in that folder).
2. Once you're in Omarchy, install the backup boot file so the firmware can always find it:
   ```bash
   sudo sed -i 's/^ENABLE_LIMINE_FALLBACK=.*/ENABLE_LIMINE_FALLBACK=yes/' /etc/default/limine
   grep -q '^ENABLE_LIMINE_FALLBACK=' /etc/default/limine || echo 'ENABLE_LIMINE_FALLBACK=yes' | sudo tee -a /etc/default/limine
   sudo limine-install --no-efi-register
   ls -l /boot/EFI/BOOT/BOOTX64.EFI     # must exist now
   ```
   The setting stays in place, so Limine updates keep the backup file current. If F9 still doesn't list Omarchy, *Boot From EFI File* always works.

### After installing Omarchy

```bash
git clone https://github.com/jacobdeanpugh/omen16-linux-fix.git
cd omen16-linux-fix
chmod +x bin/*            # in case the executable bit was lost on Windows

# 1. Look around (read-only)
bin/omen-status

# 2. Start the battery ledger
sudo bin/omen-drain install

# 3. Baseline: confirm the bug on your machine
#    charger OUT -> systemctl poweroff -> wait >= 1 hour -> power on, unlock
bin/omen-drain report     # expect ~20 W: "FAIL: dGPU still powered"

# 4. Check the firmware matches what the fix expects
sudo bin/omen-acpi-check  # all checks should PASS

# 5. Install the fix
sudo bin/omen-s5-install  # should end with "omen_s5: ready"

# 6. Measure again, once per setup you use:
#    no monitor / HDMI monitor attached / 2x USB-C attached
bin/omen-drain report     # expect ~0.3-2 W: "OK: really off"
```

### Reading `omen-drain report`

```
powered off at    hours  Wh used  avg W  fix  verdict
2026-10-20T22:00   2.00    40.00  20.00  off  FAIL: dGPU still powered  [dGPU suspended/D3cold; displays eDP-1@amdgpu]
2026-10-21T23:00   8.00     4.00   0.50  on   OK: really off  [dGPU active/D0; displays HDMI-A-1@nvidia,eDP-1@amdgpu]
```

- **avg W ≤ 3** means really off. **≥ 8** means the dGPU stayed powered. Anything in between is unclear; measure again over a longer window.
- A window only counts if all of these hold:
  - the charger was unplugged at both shutdown and boot (USB-C docks that charge the laptop count as a charger)
  - the laptop was off for at least 15 minutes
  - it was a real poweroff, not a reboot
- Longer windows are more accurate. The boot reading is taken after you unlock the disk, so the boot itself adds about 0.1–0.3 Wh.
- **Don't boot Windows between a Linux shutdown and the next Linux boot.** The ledger can't see Windows, so that time would be counted as "off".
- The off-time is measured with the hardware clock at both shutdown and boot, so it stays correct even if Windows keeps that clock in local time. A window where the clock moves backwards or jumps by weeks is rejected.
- `fix` shows whether `omen_s5` was loaded at shutdown. `displays` shows what was connected and which GPU drove it.

---

## Keeping it working

- **Kernel updates:** DKMS rebuilds `omen_s5` automatically. `bin/omen-status` shows whether it loaded.
- **BIOS updates:** run `sudo bin/omen-acpi-check` again. If HP changes the firmware tables, the fix may no longer apply, or may no longer be needed. The script flags both cases.
- **Secure Boot:** if you turn it on later (for example with `sbctl`), run `bin/omen-status` to confirm the module still loads. It may need signing.

## Uninstall

```bash
sudo bin/omen-s5-install uninstall   # module stays active until the next reboot
sudo bin/omen-drain uninstall        # the ledger file is kept in /var/lib/omen-drain/
```

---

## Trust

The installer builds a kernel module as root, so it uses only code that has been reviewed:

- **Pinned commit:** upstream [`13740806`](https://github.com/Adiakys/omen16-s5-fix/commit/13740806f876f3884963621ccc9dedef828edb5c).
- **Pinned files:** only the four files that get compiled (`omen_s5.c`, `os5f_aml.h`, `Makefile`, `dkms.conf`). Each one's SHA-256 must match `bin/omen-s5-install`, or the build is refused. Upstream's `install.sh` is not run.
- **Prebuilt table checked by hand:** `os5f_aml.h` contains the ACPI table already compiled to bytes. It was decoded byte by byte and matches `os5f.asl` exactly: 211 bytes, valid checksum, and no other methods.
- **Kernel behaviour checked against current mainline:**
  - the priorities: `SYS_OFF_PRIO_FIRMWARE` = 224 for `acpi_power_off_prepare`, `SYS_OFF_PRIO_HIGH` = 192 for `omen_s5`
  - the call order inside `kernel_power_off()`
  - the unconditional `pm_runtime_resume()` in `pci_device_shutdown()`

To move to a newer upstream version, review the diff, then update `COMMIT` and the hashes in `bin/omen-s5-install`.

## Credits

- [Adiakys/omen16-s5-fix](https://github.com/Adiakys/omen16-s5-fix): the `omen_s5` module used here.
- [paolo-de-marinis/omen-acpi](https://github.com/paolo-de-marinis/omen-acpi): the original analysis of `_PTS` → `PEGP._PS3` → `PG00._OFF` and the `NVDE` dependency.
- [December172/s5-shield](https://github.com/December172/s5-shield) and [AnxoCalvo/s5-poweroff-fix](https://github.com/AnxoCalvo/s5-poweroff-fix): the kernel-side diagnosis (`pci_device_shutdown()` resuming the GPU) and the battery-ledger measurement method.
