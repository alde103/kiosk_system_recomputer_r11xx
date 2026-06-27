# Design: kiosk_system_recomputer_r11xx

**Date:** 2026-06-26
**Author:** alde103 (aldebaran@valiot.io)
**Status:** Approved

## Goal

Fork `alde103/nerves_system_recomputer_r11xx` into a new Nerves system named
`kiosk_system_recomputer_r11xx`, and add the packages required to run a
fullscreen web browser in kiosk mode, mirroring `nerves-web-kiosk/kiosk_system_rpi4`.

## Background / Research

| System | Base | nerves_system_br | Toolchain | Kernel |
|--------|------|------------------|-----------|--------|
| `nerves_system_recomputer_r11xx` (source) | rpi4 **v2.0.4** | 1.33.9 | 13.2.0 | 6.12 |
| `kiosk_system_rpi4` **tag v2.0.4** | rpi4 **v2.0.4** | 1.33.8 | 13.2.0 | 6.12 |

Because both share the **rpi4 v2.0.4** base, the kiosk delta applies cleanly with
no version-conflict resolution.

**reComputer R11xx hardware:** Raspberry Pi CM4 (BCM2711, same SoC as RPi4), with
**1x HDMI 2.0 output** — so kiosk display is viable. Its `config.txt` already has
`dtoverlay=vc4-kms-v3d`, `gpu_mem=192`, `max_framebuffers=2` (the DRM/KMS GPU stack
Weston/COG need).

**Recomputer-specific customizations to preserve** (delta vs base rpi4 v2.0.4):
- `Config.in` + `external.mk` sourcing two custom Buildroot packages:
  `recomputer-r110x` (device tree overlays incl. TPM SLB9670) and `rtc-pcf8563w`
  (RTC kernel module via dkms).
- `config.txt`: `dtoverlay=reComputer-R110x`.
- `nerves_defconfig`: enables `BR2_PACKAGE_RECOMPUTER_R110X` and `BR2_PACKAGE_RTC_PCF8563W`;
  keeps base rpi4 packages (pigpio, rpi_userland, cairo, etc.).
- Custom `fwup.conf.eex`, `post-createfs.sh`, kernel defconfig, assets, VERSION.

**Kiosk delta (kiosk_system_rpi4 v2.0.4 vs base rpi4 v2.0.4):**
- `nerves_defconfig`: adds `WESTON`, `COG` (+ `COG_PROGRAMS_HOME_URI`), `WPEWEBKIT`
  (+ `WPEWEBKIT_MULTIMEDIA`), `LIBERATION` fonts, `LIBSOUP3_SSL`,
  `GST1_PLUGINS_BASE_LIB_OPENGL`, `ROOTFS_DEVICE_CREATION_DYNAMIC_EUDEV`; build tuning
  (`BR2_JLEVEL=4`, `BR2_CCACHE`); removes `pigpio`, `rpi_userland`, `cairo`,
  `BR2_ENABLE_DEBUG`.
- `rootfs_overlay/etc/erlinit.config`: IEx console moved from `tty1` (HDMI) to
  `ttyS0` (UART), freeing HDMI for the browser.
- `post-build.sh`: trims `usr/lib/dri` and WPEWebKit libs from `STAGING_DIR` to shrink
  the cached/hosted system artifact (does NOT affect the device rootfs).
- `linux-6.12.defconfig`: `CONFIG_HID_MULTITOUCH=m`, more UARTs, POE HAT, `PREEMPT_RT`→`PREEMPT`.
- `cmdline-a.txt`: unchanged (kernel keeps `console=tty1` + `quiet`; Weston takes the VT).

**Browser launch is out of scope for the system repo.** The system only ships the
binaries (weston/cog/wpewebkit). Starting Weston+COG is the Elixir application's job
(cf. `kiosk_example` / `recomputer_example`).

## Approved Decisions

1. **Fork mechanism:** Copy source to a new local project, create
   `alde103/kiosk_system_recomputer_r11xx` on GitHub, push.
2. **IEx console:** Move to UART (`ttyS0`), freeing HDMI for the browser.
3. **Packages:** Keep all reComputer hardware packages (pigpio/RTC/overlays) and
   ADD the browser stack (additive merge — do NOT remove pigpio/rpi_userland/cairo).

## Changes

### Buildroot / system

- **`nerves_defconfig`** (additive): add
  - `BR2_PACKAGE_WESTON=y`
  - `BR2_PACKAGE_COG=y`, `BR2_PACKAGE_COG_PROGRAMS_HOME_URI="http://localhost:4000/"`
  - `BR2_PACKAGE_WPEWEBKIT=y`, `BR2_PACKAGE_WPEWEBKIT_MULTIMEDIA=y`
  - `BR2_PACKAGE_LIBERATION=y`, `BR2_PACKAGE_LIBSOUP3_SSL=y`,
    `BR2_PACKAGE_GST1_PLUGINS_BASE_LIB_OPENGL=y`
  - `BR2_ROOTFS_DEVICE_CREATION_DYNAMIC_EUDEV=y`
  - set `BR2_NERVES_SYSTEM_NAME="kiosk_system_recomputer_r110x"`
  - keep `BR2_PACKAGE_RECOMPUTER_R110X=y`, `BR2_PACKAGE_RTC_PCF8563W=y` and base packages
- **`rootfs_overlay/etc/erlinit.config`**: uncomment `-c ttyS0`, comment out `-c tty1`.
- **`post-build.sh`**: add the `STAGING_DIR` trim line for `usr/lib/dri` + WPEWebKit libs.
- **`linux-6.12.defconfig`**: add `CONFIG_HID_MULTITOUCH=m` (touchscreen support). Leave
  the rest of the recomputer kernel config (incl. PREEMPT_RT) untouched.
- **`config.txt`**: unchanged (KMS/GPU already present).
- **`cmdline-a.txt`**: unchanged.

### Project metadata

- **`mix.exs`**: `@app :kiosk_system_recomputer_r11xx`, `@github_organization "alde103"`,
  description `"Kiosk Nerves System for reComputer R11xx (64-bit)"`. `package_files`
  keeps `Config.in`, `external.mk`, `package/` (already present in recomputer).
- **`VERSION`**: `2.0.4-recomputer-r11xx-kiosk-v1`.
- **`README.md`**: document kiosk mode — included packages, console on UART, how an app
  consumes it (`MIX_TARGET`, dep snippet), pointer to an example app.
- **`REUSE.toml`** + **`CHANGELOG.md`**: new entry describing the fork and kiosk delta.

### Repository

- Copy recomputer source (exclude `_build`, `deps`, `.git`, `*.tar.gz`, `*.log`,
  `.elixir_ls`, `.nerves`, generated `fwup.conf`) to
  `/home/alde/Documents/Elixir/Nerves/repos/kiosk_system_recomputer_r11xx`.
- `git init`, apply changes, initial commit.
- `gh repo create alde103/kiosk_system_recomputer_r11xx` (public), push `main`.

## Out of Scope

- Elixir launcher app that starts Weston+COG (separate example/library — follow-up).
- CI workflow (would use custom `X64`/`ARM64` runners per org convention) — add only if requested.
- Building full firmware here (Buildroot build is long); verification is project-load + linter.

## Verification

- `mix deps.get` resolves.
- `mix nerves_system_linter` (or `mix compile`) passes on the new project.
- Manual diff review: kiosk delta present, recomputer hardware packages intact.
