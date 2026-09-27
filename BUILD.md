# GoboLinux Game LiveCD — Build Notes & Handoff
# note  you building this iso on Arch linux 
# Gobo Build Live CD  https://github.com/gobolinux/BuildLiveCD

Everything needed to rebuild, resume, or continue testing the **GoboLinux gaming LiveCD**
(kernel **7.2.5-1 CachyOS** or 7.1.5 mainline + NVIDIA 580.159.04 + Vulkan 1.4.357.0 + Steam
with baked GE-Proton11-3 + ZFS-on-root support for the installer/boot path).

**Method in one line:** extract the stock GoboLinux 017.01 live rootfs on Arch, chroot into
it, `Compile`/`CreatePackage` the custom packages there, `RefreshLiveCD`-merge them back into
a fresh rootfs + dracut initramfs, then assemble the ISO on the Arch host (chroot has no
xorriso/isohybrid). Round 2 (Sep 22) adds the CachyOS kernel swap + OpenZFS/ZFS-root
plumbing (`step_fs` + livefix §45) and the XIVLauncher CA-import fix (§46) — all applied
automatically by `merge`/`livefix`.

---

## 1. Directory map

| Path | Purpose |
|---|---|
| `/home//Projects/BuildLiveCD/` | Repo (git HEAD `5f653bf`). Custom recipes + tooling live here. |
| `Recipes/{Linux,Nvidia,Steam,Glibc-32,Vulkan-Headers,Vulkan-Loader,Vulkan-Tools}/` | Custom Gobo recipes (see §4). `Linux/7.2.5` = CachyOS `linux-cachyos-zfs`. |
| `gobo-build/zfs45.sh` | One-shot, idempotent ZFS-root patcher (called by livefix §45): patches 5 files (90zfs dracut module-setup, GenGrubConf, WriteBoot64, BootUp, Installer). |
| `gobo-build/goboLinuxInstaller-xfs-zfs.patch`, `gobo-installer-swap-zfs.patch`, `gobo-zfs-sysvinit-support.patch` | ZFS patch artifacts (fork first, then swap; the 3rd covers the 4 SysVinit runtime files). See `PROGRESS.md` Sep 22. |
| `bin/AddProton` | Downloads a GE-Proton tarball and bakes it into `Steam/.../compatibilitytools.d/`. |
| `bin/RefreshLiveCD`, `bin/CompressAndBuildISO`, `bin/MakeInitRDTree` | Stock Gobo tooling (modified, see §6). |
| `Data/Packages-List-Game` | The game package list used for reference. |
| `Recipes/{Mutter/48.7/, GLib/2.84.4/, HarfBuzz/11.4.0/, Fribidi/1.0.16/, Pango/1.56.1/, JSON-GLib/1.10.0/, GdkPixbuf/2.42.12/, Graphene/1.10.8/, LCMS2/2.19.1/, Libei/1.3.901/, Libdisplay-Info/0.3.0/, GSettings-Desktop-Schemas/48.0/, Sysprof-Capture/48.0/, Libwacom/2.16.0/, LibInput/1.31.3/, Elogind/257.16/, LibGLVnd/1.7.0/, LLVM/19.1.7/, Mesa/25.3.6/, LibPipewire/1.4.0/, LibCanberra/0.30/, LibGudev/238/, Colord/1.4.8/, Gnome-Desktop/44.5/, GTK4/4.20.0/, LibYaml/0.2.5/, LibXmlb/0.3.29/, AppStream/1.0.6/, LibAdwaita/1.8.0/}` | Pantheon stack port — GNOME 48 foundation (29 recipes, 7 layers), **written + wired into `step_pantheon()` but NOT yet compiled**. GTK 4.20.0 (bumped from 4.18.6 for libadwaita 1.8.0's GTK >= 4.19.4). No systemd anywhere: Elogind 257.16 provides logind; boot wiring (BootUp append + PAM-header symlink) lives in `build.sh` because a recipe PostInstall's writes get discarded by the UnionSandbox. See `Recipes/pantheon-recipes.md` for build order, constraints and status. |
| `Recipes/{Wine,Wine-Lutris-GE,DXVK,Vkd3d-Proton,Heroic,Lutris,OpenSSL,Python,PyCairo,PyGObject}/`, `Recipes/Wine-Source/11.14/` | **winedev** gaming stack (see §2): Wine 11.14 (vanilla Kron4ek) + Wine-Lutris-GE 11.14 (Kron4ek staging) manifest recipes, DXVK 3.0.2 + Vkd3d-Proton 3.0.1 DLL manifest recipes, Heroic 2.22.0 (Electron + `--no-sandbox` wrapper), and the **Python 3.11 migration** chain (OpenSSL 1.1.1w → Python 3.11.12 → PyCairo 1.20.1 → PyGObject 3.44.1 → Lutris 0.5.22 + Protontricks 1.14.1 rebuild). `Wine-Source/11.14` is an optional source-build, NOT on the default ISO. |
| `Recipes/pantheon-recipes.md` | Pantheon-specific recipe tracker (GNOME 48 era) — build order, deps, verification steps, post-Mutter shells. Updated each session. |
| `/home//gobo-build/` | All build working files (NOT in git). |
| `gobo-build/GoboLinux-017.01-x86_64.iso` | Reference stock ISO. SHA256 `8cd66e3e06c865c375134d099b8b483776ef16edc6d3cd617740c36f6ba53bd3`. |
| `gobo-build/rootfs/` | Unsquashed Gobo rootfs (the chroot). |
| `gobo-build/iso-extract/` | Extracted reference ISO boot tree (isolinux/, squashfs). |
| `gobo-build/run-build.sh` | Host→chroot launcher (bind mounts + `exec chroot`). |
| `gobo-build/build.sh` | In-chroot orchestration (all build steps). |
| `gobo-build/refresh-merge.py` | RefreshLiveCD copy with the ISO-create step removed. |
| `gobo-build/finalize-iso.sh` | Host-side ISO assembly (mksquashfs + xorriso + isohybrid). |
| `gobo-build/logs/`, `gobo-build/Packages/`, `gobo-build/work/` | Build logs, package tarballs, merged rootfs. |
| `/tmp/opencode/gobo-recipes` | Upstream `gobolinux/Recipes` clone (patch reference). |

---

## 2. The custom packages (in `Recipes/`)

| Program | Version | Notes |
|---|---|---|
| **Linux** | 7.1.5 | Upstream `linux-7.1.5.tar.xz` (kernel.org). Kernel release **`7.1.5-1`** (`CONFIG_LOCALVERSION="-1"`). One Gobo patch: `01-gobohide.patch` (matches upstream 6.12.x exactly). Config: `x86_64/dot-config` (Gobo's shipped config, `CONFIG_GOBOHIDE_FS=y`). |
| **Nvidia** | 580.159.04 | Proprietary `.run` treated as a binary archive; **only the kernel modules are compiled** (`make module SYSSRC=...`) against `/Programs/Linux/Current/lib/modules/7.1.5-1/build`. Modules → `lib/modules/7.1.5-1/kernel/drivers/video/`. Blacklists nouveau + `options nvidia-drm modeset=1 fbdev=1` in `modprobe.d/nvidia.conf`. |
| **Glibc-32** | 2.43-2 | Manifest recipe unpacking Debian i386 glibc debs into `/lib32` + `/lib/ld-linux.so.2`. Dependencies: Glibc. For 32-bit Steam client/games. |
| **CMake** | 3.30.4 | Bootstrap build with **bundled** libs (no `--system-libs`), self-contained on the 2020-era chroot. Needed because the reference ISO's CMake 3.16.4 is too old for the Vulkan SDK 1.4.357.0 stack (needs ≥ 3.22.1). |
| **Vulkan-Headers** | 1.4.357.0 | CMake install. |
| **Vulkan-Loader** | 1.4.357.0 | CMake. **No Wayland WSI** (Wayland not in chroot); XCB/X11 only, `-DCMAKE_PREFIX_PATH=/System/Index`. |
| **Vulkan-Tools** | 1.4.357.0 | Same constraints as Loader. |
| **Steam** | 1.0.0.87_bin | `pre_install` unpacks the steam `.deb` data.tar.xz; unmanaged files include `etc/udev/rules.d/*`, `usr/share/applications`, icons, and **`Users/root/.local/share/Steam/compatibilitytools.d/`** where GE-Proton11-3 is baked (via `AddProton`). |
| **Wine** | 11.14 | Manifest recipe of the Kron4ek **vanilla** `wine-11.14-amd64.tar.xz` (prebuilt, no toolchain). `uncompress=no`; 5-archive `urls/.../file_md5s` (wine tar + mono/gecko MSIs); `pre_install` extracts the tarball and copies the MSIs into `share/wine/{mono,gecko}`. `symlink_options=(--conflict=overwrite)` so merged links replace the stock 8.0.1. Deps: Freetype, Fontconfig, Vulkan-Loader, LibX11, Glibc-32. |
| **Wine-Lutris-GE** | 11.14 | Same pattern, **Kron4ek staging** build (`wine-11.14-staging-amd64`); sorts after `Wine` so `--conflict=overwrite` lets staging win the merged symlinks. |
| **DXVK** | 3.0.2 | Manifest of `dxvk-3.0.2.tar.gz`: `x32`/`x64` DLLs → `lib/dxvk/{x32,x64}`. |
| **Vkd3d-Proton** | 3.0.1 | Manifest of `vkd3d-proton-3.0.1.tar.zst` (`.tar.zst` — `pre_install` pipes `zstd -dc | tar x`); DLLs + `setup_vkd3d_proton.sh` → `lib/vkd3d/`. |
| **Heroic** | 2.22.0 | Manifest of the official AppImage-extracted tar.xz → `lib/heroic`; `pre_install` renames the ELF to `heroic-bin` and writes a `bin/heroic` wrapper with `--no-sandbox` (chrome-sandbox can't be setuid on Live CD) + `heroic.desktop`/icon as unmanaged files. Deps: Vulkan-Loader, Wine, Wine-Lutris-GE, DXVK, Vkd3d-Proton. |
| **OpenSSL** | 1.1.1w | `recipe_type=configure` (non-autoconf → `override_default_options=yes`); `Configure linux-x86_64 shared --prefix=$target --openssldir=$goboSettings/ssl`; `install_target=install_sw`. Replaces the ISO's 1.1.1d (headers only, no libs). |
| **Python** | 3.11.12 | **New main interpreter** (user-directed pivot from "keep 3.8.1"). autoconf, `--enable-shared --with-system-expat --with-ensurepip=install --with-openssl=$goboIndex --enable-loadable-sqlite-extensions`; `symlink_options=(--conflict=overwrite)` moves the `python3` index symlink from 3.8.1 → 3.11.12 (3.8.1 remains as `python3.8`). 3.11 is the final 3.11; GPG-verified. |
| **PyCairo** | 1.20.1 | Meson, `-Dpython=python3.11`; emits `pycairo-1.20.1-py3.11.egg-info` (offline `install_requires` satisfaction). |
| **PyGObject** | 3.44.1 | Meson, `-Dpython=python3.11 -Dpycairo=enabled -Dtests=false` (`pycairo` is a meson **feature** option — `enabled`/`disabled`/`auto`, NOT boolean; `-Dpycairo=true` aborts meson setup); emits `PyGObject-3.44.1-py3.11.egg-info`. Resolves `dependency('py3cairo')` from PyCairo's `py3cairo.pc` (1.20.1 ≥ 1.16.0). |
| **Lutris** | 0.5.22 | `recipe_type=python`, `python_major=3.11` (0.5.21+ hard-requires Python ≥ 3.10). `pre_build` pip-installs the `install_requires` set (certifi, dbus-python, distro, evdev, lxml, pillow, pypresence, PyYAML, requests, protobuf, moddb) into the chroot system site-packages (satisfies easy_install) **and** `pip --target` into the package's own `lib/python3.11/site-packages` so they ship on the ISO. evdev builds against the ISO's kernel UAPI headers (Linux-Headers at `/usr/include`); the kernel-internal `.../lib/modules/<kver>/build/include` must NOT be added to `CPPFLAGS` (breaks glibc). |
| **Protontricks** | 1.14.1 | Rebuilt for `python_major=3.11`; vdf + `Pillow<11` bundled into its package tree the same way. Moved from `step_winetools` to `step_winedev` (must build AFTER the Python 3.11 flip). |

`Data/Packages-List-Game` documents the intended set.

### Kernel config facts (all verified, all read-only)
- `CONFIG_GOBOHIDE_FS=y` ✓ (GoboHide active)
- `CONFIG_DRM_NOUVEAU=m` (module, **not** built-in; blacklisted at runtime by nvidia.conf)
- `CONFIG_DEBUG_INFO_NONE=y` (no BTF → no pahole needed)
- `CONFIG_SYSTEM_TRUSTED_KEYS=""`, `CONFIG_MODULE_SIG_FORMAT=y` (signing not forced → unsigned nvidia modules load)
- Patch set = exactly `01-gobohide.patch`, identical to Gobo's own 6.12.7 / 6.12.16 recipes → **nothing missing**.

---

## 3. One-time host setup (already done)

```
sudo pacman -S xorriso squashfs-tools syslinux   # ISO tools on Arch host
mkdir -p /home///gobo-build
# download reference ISO, verify:
sha256sum GoboLinux-017.01-x86_64.iso   # 8cd66e3e06c865c375134d099b8b483776ef16edc6d3cd617740c36f6ba53bd3
xorriso -osirrox on -indev GoboLinux-017.01-x86_64.iso -extract / iso-extract/
unsquashfs -f -d rootfs iso-extract/gobolinux-live.squashfs
```

---

## 4. Build pipeline — files in order

### 4a. `run-build.sh` — HOST launcher (needs sudo)
Bind-mounts `/proc /sys /dev /dev/pts` (rbind) into the rootfs, bind-mounts
`/home///gobo-build` → `/mnt/gobo-build` and the repo → `/mnt/repo`, copies
`/etc/resolv.conf`, then `exec chroot rootfs /bin/bash /mnt/gobo-build/build.sh "$@"`.

**Idempotent** — safe to re-run with mounts already present (`mountpoint -q` guards).
Mounts can be left in place between runs.

```
sudo /home///gobo-build/run-build.sh [step]
```

### 4b. `build.sh` — IN-CHROOT orchestrator
Steps (`all` chains them in this order):

```
proton   # AddProton GE-Proton11-3 into Steam recipe dir (skips if already baked)
sync     # rm -rf + cp -a the 30 custom recipes from /mnt/repo into /Data/Compile/Recipes
aliens   # cp bin/Alien-NPM + bin/Alien-Cargo (this repo) into $B/work/rootfs —
         #  real files under Programs/Scripts/017-GIT/bin/ + /System/Index/bin/
         #  symlinks — so they ship on the ISO at /System/Index/bin/Alien-{NPM,Cargo}.
         #  MUST run AFTER 'merge' (merge wipes $B/work and re-extracts the base ISO;
         #  the ISO is built from work/rootfs, not the chroot's /System). The
         #  dispatcher's `for alien in Alien-*` help listing picks them up from the
         #  index. (Runtimes — node, npm, cargo, rustc — are NOT shipped; ROADMAP §0a.)
linux    # Kernel selection (select_kernel): interactive prompt 1=mainline 7.1.5
         #  2=CachyOS linux-cachyos-zfs 7.2.5-1  n=none; non-interactive runs reuse
         #  $B/.build-select. Re-stages Recipes/Linux from the repo each run, then
         #  Compile → CreatePackage → Packages/Linux--<brand>--x86_64.tar.bz2
fs       # xfsprogs + OpenZFS 2.4.4 (out-of-tree zfs.ko). Force-rebuilds via
         #  compile_forced when zfs.ko is missing for the Current kernel — run
         #  RIGHT AFTER `linux` on a kernel swap (build.sh:1998-2002).
nvidia   # Compile Nvidia 580.159.04 → CreatePackage   (needs Linux installed first)
nvdiag   # OpenCL-Headers 2026.05.29 → Clinfo 3.0.25.02.14 → VDPAUInfo 1.4 (NVIDIA diag tools)
glibc32  # Compile Glibc-32 2.43-2 → CreatePackage
wayland  # [tools first: Alien pip3 meson+ninja] Wayland → Wayland-Protocols → XKBcommon →
         #  LibInput → LibXKBfile → LibXcvt → EGLExternalPlatform → EGL-Wayland → Scdoc →
         #  SeatD → XWayland → Wlroots → Sway → Wlr-Protocols → Wayland-Utils → Wlr-Randr
         #  (each compiled + packaged; XWayland BEFORE Wlroots!)
vulkan   # [wayland first (vulkaninfo REQUIRES wayland-client), then CMake 3.30.4]
         #  Headers → Loader → Tools (each compiled + packaged; Wayland WSI ON)
steam    # Compile Steam 1.0.0.87_bin → CreatePackage
winetools # Compile Winetricks 20260125 (wine helper tools; Protontricks moved to winedev)
winedev   # OpenSSL 1.1.1w → Python 3.11.12 → PyCairo 1.20.1 → PyGObject 3.44.1 → Lutris 0.5.22
          #  → Protontricks 1.14.1 (REBUILD for 3.11; stale 3.8 tarball/tree cleared first)
          #  → Wine 11.14 → Wine-Lutris-GE 11.14 → DXVK 3.0.2 → Vkd3d-Proton 3.0.1 → Heroic 2.22.0
          #  (each compiled + packaged)
pycompat  # Ship gobo-3.8-site.pth in the Python 3.11 tarball (Gobo tooling on the ISO under
          #  python3=3.11; no recompile, CreatePackage-only). Repack skip-guarded on the .pth.
extras   # Discord 1.0.152 → Bubblewrap 0.11.0 → ECM 5.115.0 → SDDM 0.18.1
merge    # python3 refresh-merge.py <iso> work/ Packages/  (builds work/ rootfs + initramfs)
```

- **Resumable**: `packaged()` skips any program whose tarball is already in `Packages/`.
- Compile output is `tee`'d to the terminal AND `logs/<prog>.log`.
- Stdin is fed `/dev/null` to kill the `menuconfig` TUI (see §6 gotchas).
- Compile log for failures: `gobo-build/logs/<prog>.log`.

Individual steps can be invoked: `sudo .../run-build.sh linux`, `... nvidia`, etc.

### 4c. `refresh-merge.py` — IN-CHROOT merge (run by build.sh `merge` step)
Patched copy of `bin/RefreshLiveCD` **without** the `xorriso`/`isohybrid` ISO-create step.
Flow (see `main()`):
1. Mount reference ISO (tempdir), extract `isolinux/` + `gobolinux-live.squashfs` → `work/`
2. For each tarball in `Packages/`: remove prior version, extract into `work/rootfs/Programs/`,
   run `UpdateSettings -a <prog> <ver>` + `SymlinkProgram -c overwrite -u install` **inside a
   chroot of `work/rootfs`** (bind-mounts `/dev` `/proc` per call)
3. Update `System/Settings/GoboLinuxVersion` → `017.01`
4. Copy new kernel image `rootfs/Programs/Linux/Current/Resources/Unmanaged/System/Kernel/Boot/kernel`
   → `work/isolinux/kernel`; regenerate initramfs via dracut
5. `work/` is left ready for host-side ISO assembly

Initramfs: `dracut --kver <release> -m "bash kernel-modules kernel-modules-extra udev-rules base fs-lib shutdown img-lib dmsquash-live" --filesystems "squashfs iso9660" --show-modules --force`. The boot cmdline expects `root=live:LABEL=GOBOLINUX_LIVE_INSTALLER` + `rd.live.squashimg=gobolinux-live.squashfs`.

### 4d. `finalize-iso.sh` — HOST ISO assembly (no sudo)
Consumes `work/` produced by the merge step:
1. `mksquashfs work/rootfs/* → work/gobolinux-live.squashfs` (`-comp zstd -noappend -no-sparse -Xcompression-level 15`)
2. Extract the reference ISO's hybrid MBR (`dd bs=432 count=1`) → `isohdpfx.bin` (fallback `/usr/lib/syslinux/bios/isohdpfx.bin`)
3. `xorriso -as mkisofs` with `-eltorito-boot isolinux/isolinux.bin`, `-isohybrid-mbr`, `-eltorito-alt-boot -e isolinux/efiboot.img -isohybrid-gpt-basdat`, `-V GOBOLINUX_LIVE_INSTALLER`
4. `isohybrid --uefi` → **`work/gobolinux.iso`**

---

## 5. Canonical command sequence

```bash
# 1. Smoke-test the chroot (env + network):
sudo /home///gobo-build/run-build.sh test

# 2. Full build (kernel is the long step; resumable, ~1-2 h total):
sudo /home///gobo-build/run-build.sh          # = proton sync linux nvidia nvdiag glibc32 wayland vulkan steam winetools mingw winedev pycompat extras pantheon merge aliens

# 2b. Round 2 — CachyOS kernel + ZFS root (incremental, run now):
sudo /home///gobo-build/run-build.sh linux    # choose 2 = linux-cachyos-zfs 7.2.5-1
sudo /home///gobo-build/run-build.sh fs       # xfsprogs + OpenZFS 2.4.4 rebuild for 7.2.5-1
sudo /home///gobo-build/run-build.sh nvidia   # NVIDIA force-rebuild for 7.2.5-1 (auto)
sudo /home///gobo-build/run-build.sh merge    # re-merge + initramfs; auto-runs step_livefix (§40-46 incl. ZFS + XIVLauncher)

# 3. ISO assembly (host, no root):
/home///gobo-build/finalize-iso.sh             # → ~/gobo-build/work/gobolinux.iso
```

`select_kernel` chooses the brand once (persisted in `$B/.build-select`); `step_fs` and
`step_nvidia` force-rebuild the out-of-tree zfs/nvidia modules for whatever `Current`
kernel is installed (guards `zfs.ko` presence / `nvidia_modules_ok`).

Watch live: `tail -f ~/gobo-build/logs/Linux.log` (or any `<prog>.log`).

---

## 6. Gotchas / lessons learned (IMPORTANT for any AI resuming)

1. **`set -u` before sourcing `GoboPath` is fatal.** `GoboPath` references `$goboPrefix`
   before defining it (it's meant to expand to empty). Never `set -u` above `source
   /System/Index/bin/GoboPath`. (`build.sh` uses `set -o pipefail` only.)
2. **`HOME` must be `/Users/root`, and it must exist.** `/root` is a symlink → `Users/root`,
   which does **not** exist in the unsquashed rootfs (the live init creates it at boot).
   Compile's `Runner` does `mkdtemp ~/.local/Runner/...` and fails with ENOENT otherwise.
   `build.sh` does `mkdir -p /Users/root/{.local,.cache}`.
3. **The `menuconfig` TUI trap.** The Linux recipe gates `menuconfig` (and `WriteBoot64`)
   behind `[ -t 0 ]`. When the user runs from a real terminal, stdin **is** a tty, so the
   guard does NOT fire and menuconfig opens — invisible, because output is redirected.
   Fix: `build.sh` runs `Compile ... --batch </dev/null` so the guard always skips the TUI.
   Do NOT re-enable interactive prompts in these recipes.
4. **`/Data/Compile/Recipes` is a git repo with NO remote.** `UpdateRecipes` runs `git pull`
   before every compile and **always fails** ("no tracking information"). This is *protective*:
   the custom synced recipes are never overwritten. Do NOT add a remote or fix the pull —
   a successful pull would clobber the custom recipes with upstream versions.
5. **Nvidia kernel-release lookup.** There is no `/System/Kernel/Modules/Current` link in a
   chroot (that's a boot-time bind mount). The recipe derives the release with
   `ls /Programs/Linux/Current/lib/modules | grep -vE '^(Current|Settings|Variable)$' | head -1`
   → `7.1.5-1`. Modules land under `lib/modules/$kernelrelease/kernel/drivers/video`.
6. **No Wayland in the chroot.** Vulkan-Loader/Tools build with XCB/X11 WSI only
   (`-DCMAKE_PREFIX_PATH=/System/Index`). Do not re-add Wayland.
7. **Steam + Proton.** `SymlinkProgram` only installs unmanaged files listed in
   `Resources/Unmanaged`. GE-Proton11-3 must live at
   `Recipes/Steam/1.0.0.87_bin/Resources/Unmanaged/Users/root/.local/share/Steam/compatibilitytools.d/GE-Proton11-3`
   or it will NOT be on the ISO. (`AddProton` bakes it there.)
8. **Host vs chroot privilege split.** Agent/`sudo` is password-protected on the host, so the
   **user must run** `run-build.sh`. The agent (uid 1000) can do everything else: read logs,
   run `finalize-iso.sh`, QEMU tests, git. Inside the chroot everything runs as root.
9. **Chroot lacks** xorriso, syslinux, isohybrid, pahole, wayland, elogind, libxrandr.
   Hence: ISO built on host; dependencies that would fail are either absent from recipes
   or informational only (CoreUtils 8.31 vs ≥9.0 warning is non-fatal).
11. **The shipped kernel `build/` tree is incomplete for out-of-tree modules.**
   `private__copy_kernel_source()` copies a sanitized source tree, and the kernel's
   external-module checks require `include/generated/rustc_cfg`, `include/config/*` and
   `include/config/kernel.release` — none of which the copy includes. Building Nvidia
   against `/Programs/Linux/Current/lib/modules/<rel>/build` therefore fails with
   `ERROR: Kernel configuration is invalid ... include/generated/rustc_cfg` (and, because
   the kernel's `prepare` runs `pahole-version.sh`, a noisy-but-nonfatal
   `pahole.sh: line 66: exec: pahole: not found`).
     Fixes: the Linux recipe now runs `make prepare` in `$dest`, and the Nvidia recipe runs
     `$sudo_exec make -C <build> prepare` before `make module` (heals a kernel packaged by
     the old recipe without rebuilding it). `pahole` itself is *not* required for success —
     it is only consulted for a version warning.
 14. **Meson/Ninja exist but are dead symlinks.** `/System/Index/bin/{meson,ninja}` point to
     `/System/Aliens/PIP/3.8/bin/...` which only existed in the reference *build* environment,
     not the live rootfs. For Wayland/desktop builds, install them into the chroot with
     `sudo Alien --install PIP3:meson` (and `PIP3:ninja`) — build-time only, never packaged,
     so the ISO stays lean. NOTE: **`PIP3` must be uppercase** — Alien resolves `$type` verbatim
     to an `Alien-$type` plugin, and the plugin is named `Alien-PIP3`; `pip3:` fails with
     `Alien-pip3: not found`. ALSO: Alien-PIP installs the Python modules into
     `/System/Aliens/PIP/$pyver/lib/python$pyver/site-packages` but **never adds that dir to the
     interpreter's search path** (its header says PYTHONPATH or a .pth must do it — the reference
     build env exported PYTHONPATH, our bare chroot does not). `build.sh` now exports
     `PYTHONPATH=/System/Aliens/PIP/3.8/lib/python3.8/site-packages` at the top, otherwise
     `meson --version` dies on `from mesonbuild.mesonmain import main` (ImportError) right after a
     successful install. ALSO: **pin `ninja==1.11.1.2`** — the latest ninja's sdist
     `pyproject.toml` is unparseable by the chroot's pip 20.x (vendored pytoml) and has no
     cp38 wheel, so pip falls back to a build that dies in `load_pyproject_toml`; 1.11.1.2
     ships a `py3-none-manylinux2010` wheel that installs as-is on Python 3.8. See `ROADMAP.md`.
 15. **This Compile reads `meson_variables`, NOT `meson_options`.** Gobo's newer Compile
     (GitHub master) uses `meson_options` in meson recipes; the chroot's `Compile 017-GIT`
     BuildType_meson only forwards `meson_variables` + `build_variables`, so `meson_options`
     is silently ignored (recipes would build without `-D...` flags). All wayland recipes in
     this repo use `meson_variables`. Also: wlroots enables its xwayland support only if it
     finds the `Xwayland` binary at *its own build time* → `step_wayland` builds XWayland
     BEFORE Wlroots, and Sway needs `-Dtray=disabled -Dgdk-pixbuf=disabled` because there is
     no sd-bus (no systemd/logind) and no GdkPixbuf on the ISO.
12. **`modules.dep` must be regenerated AFTER the merge.** The Nvidia modules land in
   `Programs/Linux/.../lib/modules/<rel>/kernel/drivers/video/` via the package merge, but
   the Linux package's `modules.dep` was generated before Nvidia existed. `refresh-merge.py`
   now runs `/sbin/depmod <rel>` inside the merged rootfs (kver discovered from
   `Programs/Linux/Current/lib/modules`), so `modprobe nvidia-drm` works on the live ISO.
 10. **Kernel unmanaged files → `/System/Kernel/Boot`.** The Linux recipe's
     `unmanaged_files` is `$goboBoot/`, so the kernel image lands at
     `/Programs/Linux/Current/Resources/Unmanaged/System/Kernel/Boot/kernel`, which
     `refresh-merge.py` copies into `isolinux/`. Don't change that path.
 13. **`make prepare` in the shipped tree needs files the sanitized copy omits.** The kernel
     ships a sanitized source copy (`private__copy_kernel_source` copies headers/config files
     plus a few `.c` offsets files). Two categories broke out-of-tree module builds:
     - `kernel/sched/rq-offsets.c` (Linux 7.1.x added it to the prepare chain — `Kbuild:41-46`
       generates `include/generated/rq-offsets.h`); the old filter list missed it → `make prepare`
       dies with `/bin/sh: kernel/sched/rq-offsets.s: No such file or directory` at the
       `filechk_offsets` step.
     - `tools/arch` + `tools/include`: `make prepare` rebuilds objtool (`Makefile:1576`),
       which includes `-I tools/{include,arch/$(SRCARCH)/include}` (`tools/objtool/Makefile:55-58`)
       and needs `tools/arch/x86/tools/gen-insn-attr-x86.awk` to regenerate `inat-tables.c`;
       without them it fails with `No rule to make target '../arch/x86/tools/gen-insn-attr-x86.awk'`.
      Fixed twice: the Linux recipe's copy now ships `rq-offsets.c` and
      `tools/{objtool,build,lib,arch,include}` (recipe lines ~97/124), and the Nvidia recipe
      restores all of them from `linux-7.1.5.tar.xz` before `make prepare`, so an
      already-packaged kernel self-heals without a rebuild.
 16. **The Python 3.11 flip breaks Gobo's own tooling unless its 3.8 modules stay visible.**
      The `Python 3.11.12` recipe flips `/System/Index/bin/python3` → 3.11, but Gobo's Scripts
      python modules (`PythonUtils`, `GuessProgramCase`, `UseFlags`, `Alien`, `FindPackage`,
      `GetAvailable`, ...) live in the **3.8** site-packages
      (`/System/Index/lib/python3.8/site-packages` → `/Programs/Scripts/017-GIT/...`). They are
      pure-Python (loadable by 3.11), but a bare `python3` no longer searches that dir, so
      `UseFlags`/`CheckDependencies` (called by `Compile`) die with
      `ModuleNotFoundError: No module named 'PythonUtils'` and every remaining compile aborts.
      Fix (both layers, all inside `run-build.sh all`):
      - **Chroot (build-time):** `build.sh` prepends `/System/Index/lib/python3.8/site-packages`
        to `PYTHONPATH` so Compile's python helpers resolve immediately.
      - **ISO (runtime):** the `pycompat` step (new, runs after `winedev`, before `merge`)
        writes `gobo-3.8-site.pth` (content `/System/Index/lib/python3.8/site-packages`) into
        the Python 3.11 tree's own `lib/python3.11/site-packages/` and repackages the tarball
        with `CreatePackage` (no recompile). The Python recipe's `post_install` writes the same
        file for future fresh builds. Result: `Compile`/`UseFlags`/`UpdateRecipes`/etc. work on
        the booted ISO with `python3` = 3.11. Safe to re-run (guarded on the tarball containing
        the .pth). The merge step is unaffected either way (`UpdateSettings`/`SymlinkProgram`
        are bash; `refresh-merge.py` is stdlib-only).

---

## 7. Testing checklist (QEMU/KVM on the host)

```bash
# boot the hybrid ISO (UEFI or BIOS):
qemu-system-x86_64 -enable-kvm -m 4096 -smp 4 \
  -drive file=~/gobo-build/work/gobolinux.iso,format=raw,if=virtio \
  -device qemu-xhci -device virtio-vga -display sdl -net nic -net user
```

Verify in the live session:
- [ ] Boots (dracut `dmsquash-live` finds the squashfs; drops to Gobo console)
- [ ] `uname -r` → `7.1.5-1`
- [ ] `nvidia-smi` works (proprietary modules loaded; `modprobe nvidia-drm`)
- [ ] `lsmod | grep nvidia` → nvidia/modeset/drm/uvm present, nouveau **absent**
- [ ] `vulkaninfo --summary` lists NVIDIA driver as ICD
- [ ] Steam launches; Proton GE-Proton11-3 selectable in Steam → Settings → Compatibility
- [ ] `wine --version` → `wine-11.14` (vanilla) and `wine-lutris-ge --version` → staging 11.14
- [ ] `python3 --version` → `Python 3.11.12`; `python3.8` still present as `python3.8`
- [ ] `lutris --version` launches (Python 3.11 + bundled deps importable)
- [ ] `protontricks --version` runs on 3.11
- [ ] DXVK/Vkd3d DLLs present under `lib/dxvk/` + `lib/vkd3d/` on `Programs`
- [ ] Heroic launches via `bin/heroic` (no-sandbox wrapper); window appears
- [ ] 32-bit client runs (`file $(which steam)`, launch from live root)
- [ ] `/Programs` hierarchy intact (Gobo symlink layout)
- [ ] `mount | grep /lib/modules` shows the bind mount (boot-time `/System/Kernel/Modules`)
- [ ] **Boot/runtime fixes§ (2026-09-16, verified on the old ISO):**
- [ ] `sudo -u live echo ok` → 0 (sudoers root:root 0440, plug-ins setuid)
- [ ] `ssh root@127.0.0.1 -p 2222` works from first boot (ed25519 keys + `/var/empty` 0755)
- [ ] `ping -c1 10.0.2.3` + DNS resolve (`dig`/`curl`) — §12b resolv.conf + dhcpcd
- [ ] `python3 -c "import gi.repository.Gio"` → OK (§34 typelibs)
- [ ] Lutris launches (GUI) and reaches its main window (§36 datapath) — **fully working**
- [ ] `lutris` → wine runners ("lutris-ge-11.14") detected
- [ ] XIVLauncher first run: past wineboot, prefix builds fully (§35 FFXIVQuickLauncher)
- [ ] VLC opens without `qt platform plugin 'xcb'` error (§37)
- [ ] `gobonet connect <ssid>` opens Qt GUI prompt with no `pinentry`/`xcb`
      abort and no `rfkill: Permission denied` (§41 pinentry-qt Qt5 plugin path
      + `90-live-rfkill.rules` MODE=0666)
- [ ] Wine GUI under SSH → run as `live` with `DISPLAY=:0 XAUTHORITY=/run/user/1000/xauth_khByzS XDG_RUNTIME_DIR=/run/user/1000` (see `Wine.md`)
- [ ] **Round 2 (CachyOS 7.2.5-1 + ZFS root + XIVLauncher, 2026-09-22):**
- [ ] `uname -r` → `7.2.5-1` (CachyOS) — or `7.1.5-1` if mainline chosen
- [ ] `modprobe zfs` → OK; `zpool list` works (step_fs OpenZFS 2.4.4 rebuilt for 7.2.5-1)
- [ ] **Installer on a ZFS disk**: creates pool + **swap zvol**, installs EFI GRUB *with*
      `zfs` module (fork + swap patches apply post-`merge` via §45)
- [ ] Post-install boot: GenGrubConf wrote `root=zfs:<ds>` + initrd entry; pool imports
      and `udevadm settle` runs **before** `swapon` (§45 BootUp patch) → boots with
      ZFS swap active
- [ ] On the live ISO: `dmesg`/`cat /proc/cmdline` shows `zfsroot=`; `zpool status` OK
- [ ] XIVLauncher first run ONLINE: prefix CA store populated
      (`$WINEPREFIX/.gobo-cacert-imported` present), Velopack self-update + SE login +
      Dalamud reachable (§46; run as `live`, and check the root:root
      `/Programs/FFXIVQuickLauncher` write-permission watch item)

---

## 8. Current status (as of last write)

### Round 2 — Sep 22 (CachyOS kernel → OpenZFS → NVIDIA → merge; ZFS root + XIVLauncher supports)
- **CachyOS 7.2.5-1 kernel build**: gut of the compile FAILED on `make` in `fs/`:
  `fs/gobohide.c:296: error: implicit declaration of function 'strncpy'`. Kernel 7.2.x
  **removed `strncpy()` entirely** (the 7.1.5 gobohide patch's `strncpy` call was the
  last holdout — the 7.2.5 port hits an API that no longer ships a prototype).
  Fixed in the recipe patch (`01-gobohide.patch`: `strncpy(...)` → `memcpy(...)` —
  the call copies `size` raw bytes into an NLA payload, so `memcpy` is correct) and
  `compile()` now forwards extra flags to `Compile` so `step_linux` **resumes** a
  present-but-stale source tree with `--lazy` (keeps the ~90% built objects; skips
  unpack/reconfig/repatch) and **self-heals** the stale `strncpy` line in the
  extracted tree before resuming. Re-run is just `sudo ./run-build.sh linux`.
- **CachyOS 7.2.5-1 kernel build IN PROGRESS** (`run-build.sh linux`, menu **2** =
  `linux-cachyos-zfs`): `linux-cachyos-7.2.5-1.tar.gz` fetched from
  github.com/CachyOS/linux-cachyos; `Recipes/Linux/7.2.5` ships a regenerated
  `01-gobohide.patch` (28740 B — 7.2.5 API, differs from 7.1.5) and disables
  LTO_CLANG/RUST (GCC build); `CONFIG_XFS_FS=m` + `CONFIG_IKCONFIG=y/PROC` verified
  in `x86_64/dot-config`. `step_linux` re-stages the recipe and Compile re-patches
  in batch mode. Then `fs` (OpenZFS 2.4.4 zfs.ko rebuild for 7.2.5-1, else
  `compile_forced` via the zfs-module guard build.sh:1998-2002) → `nvidia`
  (`nvidia_modules_ok` force-rebuild, build.sh:566-576) → `merge`.
- **ZFS-on-root delivered** (livefix §45 → `gobo-build/zfs45.sh`): SysVinit-runtime
  90zfs module-setup, GenGrubConf `root=zfs:`/root-token/initrd/`insmod zfs` handling,
  WriteBoot64 `--add zfs`, BootUp `udevadm settle` before swapon, and installer pool +
  swap-zvol. Patches: `goboLinuxInstaller-xfs-zfs.patch` (EFI hunk retargeted to
  `grub-mkstandalone-efi --modules="... font zfs"`), `gobo-installer-swap-zfs.patch`
  (**apply after the fork patch**), `gobo-zfs-sysvinit-support.patch`. Applied to both
  `rootfs` trees with `sudo bash zfs45.sh {work/rootfs,rootfs}`; `GenGrubConf` F-test
  13/13 (`/tmp/opencode/zfs45/test_ggc.py`, `types.ModuleType` exec loader + mangled
  `_GrubConf__` helpers). Details + flow in `PROGRESS.md` (Current Status Sep 22).
- **XIVLauncher CA fix delivered** (livefix §46): empty Wine-prefix Windows cert store
  was failing every .NET/SChannel TLS call; wrapper now imports the host CA bundle
  per-prefix, once (`certutil -addstore -f root`, flag `.gobo-cacert-imported`). Also
  already applied directly in the repo recipe
  `Recipes/Gaming/FFXIVQuickLauncher/7.0.20/Recipe`. See `NOTES.md` round-2 section.
- ⚠ Watch item: Velopack in-place update writes to root-owned `/Programs/...` (0755) —
  verify as the `live` user on the new ISO.

### Round 1 (Sep 16)

- Reference ISO downloaded + verified; rootfs extracted; chroot tested.
- GE-Proton11-3 baked into Steam recipe ✓.
- Kernel **7.1.5** compiled and packaged (`gobo-build/Packages/Linux--7.1.5--x86_64.tar.bz2`, 2.4 GB).
  ⚠ The built config drifted from the shipped `dot-config`: the source `.config` predates
  the dot-config edit, so the release is `7.1.5` (not `7.1.5-1`) and
  `CONFIG_DEBUG_INFO_DWARF5=y` (not `DEBUG_INFO_NONE`). Functionally fine — the Nvidia recipe
  derives the kernel release dynamically. To get the documented `7.1.5-1` + no-debug kernel:
  delete the Linux tarball, `rm -f <chroot>/Data/Compile/Sources/linux-7.1.5/.config`, re-run
  `linux` (full rebuild, ~1 h).
- **Nvidia step was failing** ("Kernel configuration is invalid ... rustc_cfg" + pahole noise).
  Root cause: incomplete kernel `build/` tree (see gotcha 11). Fixed in the recipes; the
  Nvidia recipe self-heals the installed tree via `make prepare`, so re-running `all` resumes.
  ⚠ After the `LD=ld.bfd` fix, a second blocker appeared: `make prepare` died on the missing
  `kernel/sched/rq-offsets.c` (gotcha 13) — also fixed (recipe filter + Nvidia-side tar
  self-heal). Both fixes are synced into the chroot on the next `sync` step.
- **Nvidia is now BUILT + PACKAGED** (`Packages/Nvidia--580.159.04--x86_64.tar.bz2`): all five
  modules (`nvidia`, `-uvm`, `-modeset`, `-drm`, `-peermem`) compiled against kernel 7.1.5 and
  installed to `/lib/modules/7.1.5/kernel/drivers/video`. **Glibc-32 2.43-2** also built+packaged.
- **Vulkan blocked on old CMake**: the chroot ships CMake 3.16.4, but Vulkan-Headers 1.4.357.0
  needs ≥ 3.22.1. Fixed by adding `Recipes/CMake/3.30.4` (bootstrap build, bundled libs, based on
  the upstream 3.30.4 recipe) + a `cmake` step that `step_vulkan` runs first. `build.sh` was
  updated accordingly (and `sync` now copies the CMake recipe).
  **Wayland-Protocols bumped 1.37 → 1.44** (new `Recipes/Wayland-Protocols/1.44`): Wayland-Utils
  1.3.0's `wayland-info` requires `wayland-protocols >= 1.44`. Protocol XMLs only, so nothing else
  needs rebuilding.
- Steps not yet done: wayland (see below), vulkan (cmake → headers → loader → tools), steam,
  merge; ISO not built. `refresh-merge.py` regenerates `modules.dep` during merge (gotcha 12).
- **Wayland/Sway desktop planned** (TODO2.txt → `ROADMAP.md`): 13 recipes written into
  `Recipes/` (Wayland 1.23.1, Wayland-Protocols 1.44, XKBcommon 1.7.0, LibInput 1.26.2,
  LibXKBfile 1.1.3, LibXcvt 0.1.2, EGLExternalPlatform 1.2.1, EGL-Wayland 1.1.9, Scdoc 1.11.3,
  SeatD 0.6.4, XWayland 24.1.2, Wlroots 0.15.1, Sway 1.7) with all source URLs + md5s verified,
  and a new
  `wayland` step in `build.sh` (installs meson+ninja via Alien first; XWayland built before
  Wlroots so wlroots detects it). Not yet compiled — next `all` run builds them before vulkan.
  **EGLExternalPlatform 1.2.1 added** after EGL-Wayland failed with
  `Dependency "eglexternalplatform" not found` (the NVIDIA EGL_EXT_external_platform headers);
  built before EGL-Wayland, `-Ddatadir=lib` so the generated pc lands in `lib/pkgconfig`.
  **EGL headers patched** for EGL-Wayland: the ISO ships 2020-era EGL headers (libglvnd 1.2.0
  `eglext.h` + Mesa 19.3.3 `eglmesaext.h`) that predate `EGL_NV_stream_consumer_eglimage`
  (Mesa 21.3+/Khronos) — egl-wayland's `wayland-eglhandle.h` needs its 4 PFN typedefs + 4 tokens
  (`PFNEGLSTREAMIMAGECONSUMERCONNECTNVPROC` etc., `EGL_STREAM_CONSUMER_IMAGE_NV` 0x3373,
  `EGL_STREAM_IMAGE_{ADD,REMOVE,AVAILABLE}_NV` 0x3374-6), and `wayland-drm.c` needs
  `EGL_DRM_RENDER_NODE_FILE_EXT` from `EGL_EXT_device_drm_render_node` (0x3377, registry 2022).
  The recipe's `pre_build` appends those blocks to the build-chroot's `eglext.h`
  idempotently (one guard per extension) — build-time only; the ISO rootfs is rebuilt from the
  reference ISO + packages, so the patched header never ships.
  **EGL_WL_bind_wayland_display added** (wlroots needs `PFNEGLQUERYWAYLANDBUFFERWL` in
  `wlr/render/egl.h`): the Wlroots recipe's `pre_build` appends that extension to the same
  `eglext.h`, with `struct wl_display`/`struct wl_resource` forward declarations so GCC's
  `-Werror` "declared inside parameter list" warning doesn't fire (wlroots includes `eglext.h`
  before `wayland-server-core.h`). All other EGL symbols wlroots uses were verified present in
  the 2020-era headers.
  **XorgProto 2024.1 added** after XWayland failed configure with
  `inputproto ... Found 2.3.2 but need >= 2.3.99.1` (the ISO's XorgProto 2019.2 is too old for
  XWayland 24.1.2). Meson recipe from x.org's proto archive, `-Ddatadir=lib` (pcs → lib/pkgconfig);
  ships `inputproto 2.3.99.2` (>= 2.3.99.1 satisfied) plus every other xorgproto pc at/above the
  versions XWayland 24.1.2 requires. Built before XWayland; Compile's post-build SymlinkProgram
  links 2024.1 into `/System/Index`, superseding 2019.2 (headers are compile-time only, so the
  prebuilt ISO binaries are unaffected), and the merged package replaces 2019.2 in the ISO.
  **LibDrm 2.4.121 added** after XWayland then failed configure with `libdrm ... Found 2.4.100 but
  need >= 2.4.116` (ISO ships 2.4.100). Meson recipe from dri.freedesktop.org; era-matched (Jun
  2024), soname `libdrm.so.2` is stable so prebuilt Xorg/Mesa keep working, and XWayland 24.1.2 +
  wlroots 0.15.1 (`>= 2.4.108`) are both satisfied. `intel` driver auto-enabled (pciaccess >= 0.10
  present); man-pages/valgrind/tests disabled. Built before XWayland; merged into the ISO.
  **GCC 14 calloc-transposed-args fixed** in Wlroots 0.15.1 + Sway 1.7: GCC 14's
  `-Werror=calloc-transposed-args` rejects the old `calloc(sizeof(X), N)` idiom. Wlroots:
  `-Dexamples=false` (demo programs not needed) + `pre_build` sed on `backend/wayland/output.c`
  and `backend/libinput/tablet_pad.c`. Sway: `pre_build` sed on `swaynag/config.c` + `swaynag/main.c`.
  Verified the other queue packages (wlr-randr, wayland-utils, wlr-protocols) have no occurrences.
  **Sway 1.7 on GCC 14 additionally needs `-Wno-` flags**: two `-Wstringop-truncation` (strncpy in
  criteria.c + config.c) and one `-Wswitch` (libinput 1.26.2's `LIBINPUT_CONFIG_ACCEL_PROFILE_CUSTOM`
  enum not covered by sway 1.7's switch). Passed via `export CFLAGS="$CFLAGS -Wno-..."`
  in the recipe, **not** `-Dc_args`: Compile's `BuildType_meson` re-word-splits `meson_variables`
  on whitespace (`meson_variables=(`echo ${meson_variables[@]} ...)`) which destroys any
  multi-flag value. meson appends env CFLAGS *after* the project's `-Werror`, so they win.
  Flags: `-Wno-stringop-truncation -Wno-switch -Wno-format-overflow -Wno-format-truncation
  -Wno-dangling-pointer -Wno-use-after-free -Wno-array-bounds -Wno-address` (the last silences
  swaynag's always-true `&buffers[N] != NULL` checks, another GCC 14 `-Waddress` regression).
- **Vulkan-Tools config failure fixed**: `vulkaninfo` does `pkg_check_modules(WAYLAND_CLIENT REQUIRED
  wayland-client)` whenever the Wayland WSI is enabled (its default), so running `vulkan` standalone
  without the `wayland` step failed. `step_vulkan` now runs `step_wayland` first (Vulkan-Tools is
  self-sufficient standalone; the loader is also built with `-DBUILD_WSI_WAYLAND_SUPPORT=ON`).
  NOTE: the loader tarball produced *before* this fix lacks Wayland WSI — delete it
  (`sudo rm ~/gobo-build/Packages/Vulkan-Loader--1.4.357.0--x86_64.tar.bz2`) and re-run `vulkan`
  so it rebuilds against the Wayland stack. `lib64` installs are fine: Gobo maps
  `/System/Index/lib64 -> lib`.
- **Wine helper tools added** (`step_winetools`, runs after `steam`): `Recipes/Winetricks/20260125`
  (makefile install of the self-contained script) and `Recipes/Protontricks/1.14.1` (BuildType_python;
  `pre_build` pip-installs the `vdf` + `Pillow<11` runtime deps into the system python first, so the
  legacy `setup.py install` dependency step finds them satisfied). Wine recipe bumped **11.0 → 11.14**
  (verified tarball `wine-11.14.tar.xz`, md5 `3e9bbfd8...`); the "GCC too old" note was stale — the
  chroot ships **GCC 14.2.0**. dxvk/vkd3d stay out of the build: Proton bundles both, standalone
  Wine 11 compiles vkd3d in (WoW64 covers the 32-bit side), and winetricks can install prebuilt
  DXVK release binaries (no MinGW toolchain needed).
- **Wayland companion tools** (`Recipes/Wlr-Protocols/a741f0a`, `Recipes/Wlr-Randr/0.5.0`,
  `Recipes/Wayland-Utils/1.3.0`) added to the end of `step_wayland`. wlr-protocols is a makefile
  install whose default `check` target validates XMLs with wayland-scanner (build after Wayland);
  wlr-randr bundles its own XML (`-Dwerror=false`); wayland-utils ships `wayland-info` with the
  `drm` feature disabled (ISO libdrm 2.4.100 < required 2.4.109).
- **Desktop extras added** (`step_extras`, runs after `winetools`): `Recipes/Discord/1.0.152`
  (manifest recipe of the official tar.gz — installs the `discord` launcher + `updater_bootstrap`
  into `bin/`; the client itself downloads to `~/.config/discord/` on first run),
  `Recipes/Bubblewrap/0.11.0` (meson, `-Dman/-Dselinux` off + tests off; Flatpak's sandbox
  backend — needs LibCap, on the ISO), `Recipes/Extra-CMake-Modules/5.115.0` (KDE cmake modules,
  `find_package(ECM)` dep of SDDM), `Recipes/SDDM/0.18.1` (Qt 5.14.1-era QML login manager;
  `-DNO_SYSTEMD=ON -DUSE_ELOGIND=OFF -DENABLE_JOURNALD=OFF`; PAM on). All tarball URLs + md5s
  verified; `CMAKE_INSTALL_LIBDIR=lib` so libs land in Gobo's `lib/` tree.
  Flatpak/portal stack (OSTree, Polkit, DConf, XDG-Desktop-Portal) and AppImageKit remain
  optional follow-ups on top of bubblewrap.
- **winedev gaming stack + Python 3.11 migration added** (`step_winedev`, runs after `mingw`,
  before `extras`): recipes WRITTEN + verified for Wine 11.14 (vanilla) + Wine-Lutris-GE 11.14
  (staging), DXVK 3.0.2, Vkd3d-Proton 3.0.1, Heroic 2.22.0, OpenSSL 1.1.1w, Python 3.11.12
  (GPG-verified), PyCairo 1.20.1, PyGObject 3.44.1, Lutris 0.5.22, and the Protontricks 1.14.1
  3.11 rebuild. `build.sh` synced the 11 new recipe dirs, Protontricks moved out of
  `step_winetools` into `winedev` (with stale-3.8 tarball/tree cleanup so `packaged()` doesn't
  skip it), and the `all` target chains `winedev` before `extras`/`merge`. **None compiled yet.**
  Known risk: flipping the chroot's `python3` → 3.11 after the Python install — meson/ninja are
  shebang-anchored to `/usr/bin/python3.8` and `refresh-merge.py` is stdlib-only, so they keep
  working; all new recipes target `python3.11` explicitly.
- **First merge DONE but final ISO NOT yet built**: `refresh-merge.py` completed (04:38/04:41
  timestamps; `work/rootfs` + initramfs ready; the last 4 log lines are the merge's final
  actions, not a hang). The host-side `finalize-iso.sh` (mksquashfs zstd L15 → xorriso →
  isohybrid → `work/gobolinux.iso`) has NOT run. ⚠ Re-running it must wait for the NEXT merge
  (which overlays the winedev packages, incl. the still-missing Heroic/Lutris/OpenSSL/Python
  archives).
- **Pantheon foundation wired into `step_pantheon` (26 recipes / 7 layers), none compiled yet**:
  topo-sorted from each recipe's `Resources/Dependencies`; chain validator confirms 107
  compile/package pairs with no dupes/order violations, and every recipe is covered by
  `step_sync`. Adds LibInput 1.31.3, Elogind 257.16, LibGLVnd 1.7.0, LLVM 19.1.7,
  Mesa 25.3.6, LibGudev 238 to the foundation. Mutter 48.7 deps verified against the real
  meson.build (wayland-protocols ≥ 1.41, not the 1.48 previously claimed; libwacom enabled).
  See `Recipes/pantheon-recipes.md` and ROADMAP §7. First build = `run-build.sh all`
  (long LLVM/Mesa compiles).
- **Elogind boot/PAM wiring fixed in `build.sh`**: a recipe `Resources/PostInstall` cannot
  write to `$goboSettings/BootScripts/BootUp` — `Run_PostInstall` runs it in a UnionSandbox
  whose union list includes `/System`, so the write lands in the rw overlay and is discarded.
  PostInstall deleted; the idempotent BootUp append now runs in `build.sh` after
  `package Elogind 257.16`. Likewise the ISO's Linux-PAM ships headers under `include/security`
  but never links them, so `build.sh` symlinks
  `/System/Index/include/security → /Programs/Linux-PAM/Current/include/security`
  before Elogind (needed for `pam_elogind`).
- **Alien-NPM/Cargo shipping fixed**: the plugins must be written into `$B/work/rootfs` (the
  merged tree `finalize-iso.sh` squashes), NOT the chroot's live `/System`. `step_aliens`
  rewritten accordingly and moved to run AFTER `merge` in `all` (merge wipes `$B/work`).
  Result on the ISO: `/System/Index/bin/Alien-{NPM,Cargo}` → `/Programs/Scripts/017-GIT/bin/…`.
- **Python 3.11 flip broke Compile — now fixed** (gotcha 16): after `Python 3.11.12` moved the
  `python3` index symlink to 3.11, Gobo's Scripts modules (PythonUtils/GuessProgramCase) were
  invisible → `UseFlags`/`CheckDependencies` `ModuleNotFoundError` → PyCairo/Wine (and any
  remaining compile) aborted. Fixed in `build.sh`: `PYTHONPATH` gains
  `/System/Index/lib/python3.8/site-packages` (chroot) + new `pycompat` step ships
  `gobo-3.8-site.pth` in the Python 3.11 package (booted ISO) + Python recipe `post_install`
  for fresh builds. `step_winedev`'s stale-Protontricks cleanup is now `&&`-chained so it can't
  run after a failed Python chain. Next `all` resumes at PyCairo → PyGObject → Lutris →
  Protontricks(3.11 rebuild) → Wine → Wine-Lutris-GE → DXVK → Vkd3d-Proton → Heroic →
  `pycompat` → extras → pantheon → merge → aliens → `finalize-iso.sh`.
- **VM runtime testing round (2026-09-14/16) — all boot/runtime bugs found on the
  live ISO are FIXED and baked into the `livefix` path** (`run-build.sh livefix`
  → `build.sh step_livefix` → `gobo-build/apply-live-fixes.sh`, idempotent).
  The build pipeline itself was stable; every fix below is a *live-ISO* fix that
  `merge`/`livefix` re-applies. New sections added:
  - **§12b resolv.conf DNS** — old ISO's `/etc/resolv.conf` was 0 bytes; bake
    `nameserver 10.0.2.3` (QEMU user-net DNS) + `1.1.1.1`; dhcpcd wired into boot.
  - **§32 OpenSSH** — boot ssh-keygen used `-t rsa1` (unknown key type) → no host
    keys → sshd kex-reset loop. Now `-t ed25519`; host keys baked root-owned.
  - **§34 typelibs** — active GI tree (1.84.0 over 1.62.0) lost
    Gio/GLib/GObject/GModule-2.0.typelib; restore from base rootfs cache →
    `gi.repository.Gio` imports (Lutris needs it).
  - **§35 FFXIVQuickLauncher WINEPREFIX** — wrapper ran wineboot with no prefix
    dir; recipe gained `mkdir -p "$WINEPREFIX"` + §35 sed-patches ISO wrappers.
    ⚠ §35 originally scanned `Programs/XIVLauncher/*` but the program installs
    as **`FFXIVQuickLauncher`** → fix silently no-opped; now scans both names.
  - **§36 Lutris data path** — `bin/lutris` looks in `<lib>/lutris/share/lutris`
    (pip tree lacks it); §36 links `lib/lutris/share` → program `share`. Lutris
    fully working.
  - **§37 VLC Qt xcb plugin** — `qt.conf` (Prefix=/Programs/Qt/5.15.2) beside the
    VLC binary (same trick as SDDM greeter §1).
  - **Post-chown re-apply** (merge-chown sweep re-owns everything to uid 1000):
    setuid helpers + sudo plug-ins → root:root; `sudoers` → root:root **0440**
    (`sudoers is owned by uid 1000 should be 0`); `/var/empty` → root:root 0755
    (sshd privsep); OpenSSH host keys → root:root 600/644.
  - **Wine GUI = display creds over SSH, NOT rendering** — root SSH env has no
    DISPLAY/XAUTHORITY so wineboot halts (prefix stuck, empty system32). As
    `live` with the Plasma session's `DISPLAY=:0 XAUTHORITY=/run/user/1000/
    xauth_khByzS XDG_RUNTIME_DIR=/run/user/1000`, `wineboot -u` completes
    (1020 MB prefix, 853 system32 files) and `wine notepad.exe` maps windows
    even on llvmpipe. libEGL dri2 warnings = VM-only, real hardware unaffected.
  - **winetricks hang** — two concurrent wine commands deadlocked on one prefix
    (not a packaging bug); test rule: one wine at a time per prefix.
  - Full narrative: `PROGRESS.md` (Current Status Sep 16) + `Wine.md` (new) +
    `gobo-build/NOTES.md` (top section).

## 9. Recovery / resume

- Any failed step: fix the recipe/script, then re-run
  `sudo /home///gobo-build/run-build.sh` — completed steps are skipped
  (tarballs present) and the failed step is retried.
- Interrupted kernel compile is fine: Compile reuses `/Data/Compile/Sources/linux-7.1.5`
  (config already produced). If it ever complains about stale state:
  `sudo rm -rf <rootfs>/Data/Compile/Sources/linux-7.1.5` inside the chroot and re-run.
- Mounts left up are reusable; `run-build.sh` is idempotent. To tear down:
  `sudo umount -R /home///gobo-build/rootfs/{proc,sys,dev} ...`.
- `refresh-merge.py` reads the reference ISO each merge run (fresh `work/` each time).

