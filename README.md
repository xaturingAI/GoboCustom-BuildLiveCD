# BuildLiveCD

This package contains utilities used to create the LiveCD environment, consisting of three major scripts: `UpdateEnvironment`, `CompressAndBuildISO`
as well as `RefreshLiveCD`.

We support two methods of ISO generation:
The first method generates a new ISO
from *scratch*, while the second method allows you to iterate upon an *existing*
ISO. 

> [!NOTE]
> All commands below are supposed to be run as root.

## Method 1

* **`UpdateEnvironment`**: This script fetches + compiles the ISOLINUX
  bootloader and the BusyBox package that lives in the LiveCD's initrd. It also
  fetches a copy of GoboLinux'
  [`InitRDScripts`](https://github.com/gobolinux/InitRDScripts) project, which
  is later packaged next to BusyBox in the initrd. The GoboLinux logo shown when
  booting the ISO is also handled here. This script converts the logo from PPM
  to LSS16 (the actual format understood by ISOLINUX.)

* **`CompressAndBuildISO`**: This script performs the automated generation of
  the LiveCD tree. It is divided in 4 different stages:

    1. ***ROLayer:*** given a list of GoboLinux binary packages and an empty
  target directory, this stage creates a new root filesystem tree (including
  `/System/Settings`, Aliens, and the legacy symlinks under `/`) and
  uncompresses all packages under `/Programs`. The default target directory is
  `Output/ROLayer`.

    2. ***SquashFS:*** creates a set of squashfs images from the *ROLayer*. The
  binary packages put on each squashfs image are selected according to the
  contents of the files at `BuildLiveCD/Data/Packages-List-*`. The generated
  squashfs files are saved under `Output/ISO`.

    3. ***InitRD:*** creates an initrd image by merging the `InitRDScripts` and
  `BusyBox` packages fetched earlier by `UpdateEnvironment`. The image, a
  compressed RAM filesystem, is stored as `Output/ISO/isolinux/initrd` once it's
  been prepared.

    4. ***ISO:*** this last stage runs *mkisofs* on `Output/ISO` (to produce an
  ISO file) and makes that ISO file hybrid so it boots when copied to a USB mass
  storage device. The output file is saved as `Output/GoboLinux-NoVersion.iso`.

## Method 2

* **`RefreshLiveCD`**: This script provides an alternative, more simplistic
  approach by iterating upon an existing ISO as a base. The first time it is
  called, the script extracts all squashfs images from a reference ISO to a
  given work directory. When called a second time, it can take an extra
  argument: a path with a collection of tarballs (GoboLinux packages, in
  *.tar.bz2* format). The script will then update the old versions with the new
  ones and will regenerate an ISO.

> [!IMPORTANT]
> Ensure that kernel module *loop* is loaded: `sudo modprobe loop`.

  *Usage:*
  ```
  # RefreshLiveCD <ISO_image> <work_dir> (<package_dir>)
  ```

> [!WARNING]
> Your currently symlinked kernel *has* to match the one found on the ISO –
> else initramfs generation will fail! In case of mismatch, you can supply
> your current/desired kernel via `<package_dir>` as a package.

## Gaming Live CD

This tree carries recipes for a gaming live environment on top of the stock
GoboLinux 017 base:

- **Linux 7.1.5 / 7.2.5** (`Recipes/Linux/7.1.5` + `Recipes/Linux/7.2.5`): the Gobo
  kernel recipe with the `gobohide` patch ported (7.1.5 `CONFIG_LOCALVERSION="-1"` →
  `7.1.5-1`; 7.2.5 = **CachyOS `linux-cachyos-zfs`**, same patch regenerated for the
  7.2.5 API, LTO/RUST disabled). Choice persisted via `select_kernel`
  (`run-build.sh linux` menu 2). **ZFS-on-root** added post-build by livefix §45
  (`zfs45.sh`): SysVinit 90zfs module-setup, GenGrubConf `root=zfs:` support, EFI
  `grub-mkstandalone-efi --modules=... zfs`, udev-settle-before-swapon (XFS + OpenZFS
  modules ship with the kernel; OpenZFS itself is an out-of-tree rebuild in `step_fs`).
- **Nvidia 580.159.04** (`Recipes/Nvidia/580.159.04`): binary `.run` driver.
  The userspace (64+32-bit libs, Xorg driver, Vulkan/OpenCL ICDs, GLVND,
  firmware, tools) is extracted as-is; only the kernel modules are compiled,
  against the `Linux/7.1.5` package. `nouveau` is blacklisted and
  `nvidia-drm` modeset is enabled via shipped `/etc/modprobe.d/nvidia.conf`.
- **Steam** (`Recipes/Steam/1.0.0.87_bin`): the official launcher extracted
  from `steam_latest.deb`, including the 32-bit client bootstrap. udev rules,
  desktop entry and icons are installed into the system root. Note that
  `steam_latest.deb` does *not* bundle the scout runtime: Steam downloads it
  on first run (see the offline baking note below).
- **Proton** (pre-baked into the Steam package): a prebuilt Proton
  redistributable is unpacked to
  `/Users/root/.local/share/Steam/compatibilitytools.d/` by `bin/AddProton`
  (default: `GE-Proton11-3` from GloriousEggroll's GitHub releases, verified
  against its sha512). Valve's own Proton 11 releases are source-only, so the
  prebuilt GE-Proton distribution is used instead.
- **Vulkan-Headers/Loader/Tools 1.4.357.0**: SDK recipes so the loader
  (`vulkaninfo`, ICD discovery) is present; the NVIDIA ICD provides the
  hardware driver. `vkcube` is disabled (BUILD_CUBE=OFF) since it would need
  GLFW.
- **Glibc-32 2.43-2**: minimal 32-bit base (loader + glibc + libgcc_s +
  libstdc++ 6.0.33) for the 32-bit Steam client and 32-bit games, extracted
  from Debian i386 packages. The Steam client is ELF32 and needs
  `/lib/ld-linux.so.2`; NVIDIA's 32-bit libs land in the same `lib32` tree.
- **Wine 11.0** (`Recipes/Wine/11.0`): optional source recipe using the new
  WoW64 mode (`--enable-archs=i386,x86_64`) -- no 32-bit Unix libraries at
  runtime. The stock Gobo toolchain cannot build it; it is kept only as a
  documented alternative to the prebuilt Proton used with Steam.
- **CMake 3.30.4** (`Recipes/CMake/3.30.4`): the 017.01 base ships CMake
  3.16.4, too old for the Vulkan SDK 1.4.357.0 stack (needs >= 3.22.1). Built
  from source with bundled libs. Compiled automatically before the Vulkan
  packages.

> **Current runtime state (verified on the round-1 ISO 2026-09-16; round 2 building
> Sep 22):** the gaming stack is fully working after the VM testing round — Lutris
> (all issues fixed), XIVLauncher/FFXIVQuickLauncher first-run prefix init, VLC, sudo,
> OpenSSH, and boot-time DNS. All fixes are idempotent sections of
> `apply-live-fixes.sh` applied by `run-build.sh merge`/`livefix`. Round 2 adds
> ZFS-on-root (§45), the CachyOS 7.2.5-1 kernel swap, and the XIVLauncher TLS fix
> (§46, empty Wine cert store — prefix CA import now happens on first run). See
> [`PROGRESS.md`](PROGRESS.md) (Current Status Sep 22), [`Wine.md`](Wine.md),
> and `gobo-build/NOTES.md` for the full narrative.

### Wayland / desktop / Flatpak (planned)

Tracked in [`ROADMAP.md`](ROADMAP.md) (the live list of packages to write recipes for,
compile, and merge onto the ISO). Summary of the plan:

- **Build tooling:** Meson + Ninja via Alien (`PIP3:meson`, `PIP3:ninja` — **PIP3 uppercase**, the
  plugin is `Alien-PIP3`) -- build-time only,
  auto-installed by `step_tools` in `build.sh`; not shipped on the ISO.
- **Wayland stack (recipes already written, in `Recipes/`):** Wayland 1.23.1,
  Wayland-Protocols 1.37, XKBcommon 1.7.0, LibInput 1.26.2, LibXKBfile 1.1.3, LibXcvt 0.1.2,
  EGL-Wayland 1.1.9 -- compiled by the new `step_wayland`, which runs before `vulkan`.
- **Desktop:** **Sway 1.7** (recipe in repo) on Wlroots 0.15.1 + SeatD 0.6.4 + XWayland
  24.1.2 -- a full Wayland desktop (XWayland is built *before* Wlroots so wlroots detects
  it). Wayfire 0.7.2 is a plausible second compositor. XFCE (stale recipes), Plasma (no
  Qt6), Hyprland/Cosmic (Rust/GTK dep trees) remain deprioritized.
- **Pantheon (ACTIVE workload):** elementary OS 8 desktop (gala → libmutter) — GNOME 48
  foundation (26 recipes / 7 layers) written and wired into `step_pantheon`, none compiled
  yet; sysvinit-safe (Elogind, no systemd). See [`Recipes/pantheon-recipes.md`](Recipes/pantheon-recipes.md)
  and ROADMAP §7.
- **After Wayland:** rebuild Vulkan-Loader/Tools with `-DBUILD_WSI_WAYLAND_SUPPORT=ON`.
- **Later phases:** lib32-wayland/libdecor/libva, Flatpak (OSTree+portals on top of the now-done
  Bubblewrap), AppImage, SDDM login config.

### Build order

1. Build and install `Linux` first. `select_kernel` picks the brand: menu **2** =
   CachyOS `linux-cachyos-zfs` 7.2.5-1 (round 2), menu **1** = mainline 7.1.5. The
   `01-gobohide.patch` is applied automatically by Compile.
2. Immediately after a kernel swap: `fs` (xfsprogs + **OpenZFS 2.4.4** — force-rebuilds
   the out-of-tree `zfs.ko` for the new `Current` kernel) and `nvidia` (580.159.04
   modules force-rebuild for the new kernel). Both are guard-driven in `build.sh`.
3. Build `Glibc-32`, `Vulkan-Headers`, `Vulkan-Loader`, `Vulkan-Tools`.
4. Build `Nvidia` — its modules compile against the installed kernel
   (`/System/Kernel/Modules/Current/build`), so it must come after Linux.
5. Run `bin/AddProton` to pre-bake a Proton redistributable into the Steam
   recipe, then build `Steam`.
6. `merge` (re-merges packages + regenerates initramfs and **auto-runs the livefix
   stack**, incl. §45 ZFS-root and §46 XIVLauncher CA-import), then
   `finalize-iso.sh` on the host.
7. Create binary packages with `CreatePackage` and put them in the `Packages`
   directory (see `Data/Packages-List-Game`).
8. Run `UpdateEnvironment` then `CompressAndBuildISO`, which calls
   `BuildRoot` with every `Data/Packages-List-*`. BuildRoot dies on any
   listed-but-absent package.

### Notes

- The Steam client downloads its scout runtime on first run; for an offline
  CD, pre-bake `~/.local/share/Steam` into the squashfs image before building
  the ISO.
- Proton is a compatibility tool, not a Gobo package; it is baked into the
  Steam package at `/Users/root/.local/share/Steam/compatibilitytools.d/`
  (see `bin/AddProton`) and is selected per-game in Steam's Steam Play
  settings.
- `CompressAndBuildISO` regenerates the kernel module dependency database
  (`depmod`) on the finished RO layer so `modprobe` finds the NVIDIA modules.
- Live CD boot requires the kernel to have `BLK_DEV_RAM`, `CRAMFS` and
  `SQUASHFS` built in (`=y`); the shipped `x86_64/dot-config` already does.
- The NVIDIA recipe installs its modules under `/lib/modules/<kernelrelease>`
  (not `/System/Kernel/Modules`, which is only a mountpoint covered by the
  `/lib/modules` bind mount at boot) and ships an Xorg config without a
  hard-coded `BusID`.
