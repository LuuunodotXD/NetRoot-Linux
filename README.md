# NetRoot Linux

NetRoot Linux is a minimal, bootable Linux system built entirely from scratch —
custom kernel, musl libc, BusyBox userland, and BusyBox init. No systemd, no
glibc, no package manager, no bloat. Built to be smaller and lighter than
Tiny Core Linux while remaining fully functional (shell, networking, login).

Two architectures are supported: **amd64** (x86-64) and **i686** (32-bit x86),
each with its own kernel, BusyBox build, and bootable ISO — allowing NetRoot
to run on hardware ranging from modern 64-bit machines down to late-90s
Pentium II / Celeron-class systems.

---

## Highlights

- Fully static BusyBox userland linked against musl libc — no dynamic linker,
  no shared libraries, minimal attack surface.
- Custom `tinyconfig`-based kernel, hand-configured with only the drivers and
  features actually needed (no generic distro kernel bloat).
- BusyBox init (not systemd/sysvinit/OpenRC) — a single static binary handles
  init, login shell spawning, and process supervision via `/etc/inittab`.
- Boots as a live ISO (isolinux, legacy BIOS) or as a persistent installation
  on real disk (extlinux, MBR).
- DHCP networking out of the box via `udhcpc`.
- Minimal user management with `login`/`getty`/`shadow` support.
- Verified working on real vintage hardware (Celeron 266 "Covington", 1998),
  not just emulation.

---

## Requirements

### amd64 build

- **CPU**: any x86-64 processor, compiled for the generic `x86-64` baseline
  (no CPU-specific instruction sets required — runs on anything from a 2003+
  64-bit CPU up).
- **RAM**: **32 MB minimum** to boot. 64 MB+ recommended for comfortable
  networking and multitasking.
- **Storage**: ISO is approximately 3 MB (kernel + initramfs together are
  ~2 MB compressed).
- **Boot**: Legacy BIOS (isolinux). No UEFI support at this time.

### i686 build

- **CPU**: any i686-compatible x86 processor (Pentium Pro / Pentium II /
  Celeron-class and newer, or any modern CPU in 32-bit mode).
- **RAM**: **24 MB minimum** to boot reliably (23 MB boots inconsistently in
  QEMU; 24 MB is the confirmed stable floor). 32 MB+ recommended.
- **Storage**: similar footprint to amd64, slightly smaller due to 32-bit
  pointer/structure sizes.
- **Boot**: Legacy BIOS (isolinux). Verified on real 1998-era hardware
  (Award BIOS, Celeron 266).

> The RAM difference between architectures (24 MB vs 32 MB) is the expected
> structural overhead of 64-bit pointers and kernel data structures — not a
> configuration difference. Both builds share the same kernel version,
> BusyBox version, and feature set.

---

## Components

| Component     | Version  | Notes                                            |
|---------------|----------|---------------------------------------------------|
| Linux kernel  | 6.6.144  | LTS, `tinyconfig` base + hand-picked options       |
| BusyBox       | 1.36.1   | Static, musl-linked, provides all userland tools   |
| musl libc     | latest   | Built via `crossdev` cross toolchain (Gentoo)      |
| isolinux      | syslinux | Legacy BIOS bootloader for the live ISO            |
| extlinux      | syslinux | Bootloader for persistent disk installs            |

All builds were produced using a Gentoo host system with `crossdev` for
cross-compilation (`x86_64-linux-musl` and `i686-linux-musl` targets).

---

## Quick Start — Boot the ISO

### amd64

```sh
qemu-system-x86_64 -cdrom netroot-linux-amd64.iso -display sdl -m 64
```

Serial console (output in terminal instead of a graphical window):

```sh
qemu-system-x86_64 -cdrom netroot-linux-amd64.iso -nographic -m 64
```

### i686

```sh
qemu-system-i386 -cdrom netroot-linux-i686.iso -display sdl -m 32
```

```sh
qemu-system-i386 -cdrom netroot-linux-i686.iso -nographic -m 32
```

`qemu-system-x86_64` can also run the i686 kernel directly, but
`qemu-system-i386` more closely emulates genuine 32-bit-only hardware.

---

## Login

```
root     : root
netroot  : netroot
```

`root` has full privileges. `netroot` is an unprivileged standard user, home
directory `/home/netroot`.

Both users share `PATH=/sbin:/usr/sbin:/bin:/usr/bin` via `/etc/profile`, so
all BusyBox applets (including those normally under `/sbin`) are available
regardless of which user is logged in.

---

## Networking

Networking is not enabled automatically by default (kept manual to avoid
delaying boot when no network is present). To bring it up:

```sh
ip link set eth0 up
udhcpc -i eth0 -s /etc/udhcpc.conf
```

Test connectivity:

```sh
ping -c3 8.8.8.8
wget -O - http://example.org
```

Supported network hardware (kernel drivers included): Intel E1000/E1000E,
Realtek 8139/8169, AMD PCnet32, NE2000 PCI, nForce (forcedeth), Atheros
L1/L1E, and VirtIO-net for QEMU/KVM.

---

## Persistent Installation (disk, not live ISO)

NetRoot can be installed to a real disk instead of running as a live/volatile
ISO. The disk-based build uses `extlinux` installed to the MBR, with the
kernel booting directly against an ext4 (or ext2, for the BusyBox-only
installer path) root partition — no initramfs required once installed.

General outline (see build notes for exact commands):

1. Partition the target disk (`fdisk`, single primary partition, bootable flag set).
2. Format with `mkfs.ext4` (or `mkfs.ext2` if only BusyBox's built-in mkfs is available).
3. Copy the full rootfs (BusyBox tree + `/etc` config) onto the partition.
4. Install `extlinux` to `/boot` on the target partition.
5. Write the syslinux MBR bootstrap (`mbr.bin`) to the disk's first sector.
6. Boot directly with `root=/dev/sdX1` on the kernel command line — no
   `rdinit=`/initramfs needed.

Kernel requirements for disk installs (beyond the live-ISO config):
`CONFIG_EXT4_FS`, `CONFIG_MSDOS_PARTITION`, `CONFIG_ATA`, `CONFIG_ATA_PIIX`,
`CONFIG_SATA_AHCI` (for modern SATA), `CONFIG_BLK_DEV_SD`, `CONFIG_SCSI`.

---

## Build From Source

### 1. Toolchain

Built with Gentoo's `crossdev`, targeting musl:

```sh
emerge -av sys-devel/crossdev
crossdev --target x86_64-linux-musl --ov-output /var/db/repos/crossdev
crossdev --target i686-linux-musl --ov-output /var/db/repos/crossdev
```

> Important: if your host's global `CFLAGS` include `-march=native` (common
> on Gentoo), it leaks into the musl/gcc cross toolchain build and produces
> binaries using CPU-specific instructions (AVX, etc.) that will crash with
> "invalid opcode" on other CPUs or generic QEMU CPU models. Pin the cross
> toolchain to generic flags via `/etc/portage/package.env` before building:
>
> ```sh
> # /etc/portage/env/generic-march.conf
> CFLAGS="-O2 -march=x86-64 -mtune=generic -pipe"
> CXXFLAGS="${CFLAGS}"
> ```
> ```sh
> # /etc/portage/package.env/cross-musl
> cross-x86_64-linux-musl/* generic-march.conf
> ```
> For i686, omit `-march` entirely rather than setting `-march=i686` — some
> binutils/gcc host-side configure checks combine badly with host `-march`
> flags and CET (`-fcf-protection`) on x86-64 hosts. Just use `-O2 -pipe`.

### 2. Kernel

```sh
make ARCH=x86_64 tinyconfig    # or ARCH=i386 for the 32-bit build
```

Then hand-enable the required options — `tinyconfig` strips almost
everything, including things that are easy to assume are default:

- `CONFIG_BINFMT_ELF`, `CONFIG_BINFMT_SCRIPT` — **without these, nothing can
  execute at all**, not even statically-linked ELF binaries. This is the
  single most common cause of "No working init found" panics.
- `CONFIG_TTY`, `CONFIG_VT`, `CONFIG_SERIAL_8250` (+ `_CONSOLE`) — without
  `CONFIG_TTY`, no console works at all, serial or otherwise.
- `CONFIG_BLK_DEV_INITRD`, `CONFIG_SHMEM`, `CONFIG_TMPFS`, `CONFIG_DEVTMPFS`
  (+ `_MOUNT`).
- `CONFIG_MULTIUSER` — without this, `setgroups()` returns `ENOSYS` and
  non-root logins fail with "can't set groups: Function not implemented".
- `CONFIG_PRINTK`, `CONFIG_BUG` — not strictly required to boot, but without
  them, failures are completely silent (no kernel messages at all), making
  debugging nearly impossible.
- Networking: `CONFIG_NET`, `CONFIG_INET`, `CONFIG_PACKET`, `CONFIG_UNIX`,
  `CONFIG_NETDEVICES`, `CONFIG_NET_CORE`, plus NIC drivers as needed.
- Disk (persistent installs only): `CONFIG_EXT4_FS`, `CONFIG_MSDOS_PARTITION`,
  `CONFIG_ATA`/`CONFIG_ATA_PIIX`, `CONFIG_SATA_AHCI`, `CONFIG_BLK_DEV_SD`.

### 3. BusyBox

```sh
make defconfig CROSS_COMPILE=x86_64-linux-musl-
```

Then:

```sh
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
sed -i 's/CONFIG_TC=y/# CONFIG_TC is not set/' .config   # tc.c fails to build
                                                            # against musl's
                                                            # trimmed kernel
                                                            # headers
make -j$(nproc) CROSS_COMPILE=x86_64-linux-musl- \
  CC="x86_64-linux-musl-gcc -march=x86-64 -mtune=generic"
```

For login support, also enable: `CONFIG_LOGIN`, `CONFIG_GETTY`,
`CONFIG_CTTYHACK`, `CONFIG_FEATURE_SHADOWPASSWDS`, `CONFIG_CRYPTPW`.

`CONFIG_CTTYHACK` matters specifically for interactive shells spawned
directly by init without a proper `getty`/`login` — without it (or without
`/sbin/cttyhack /bin/sh` in `/etc/inittab`), job control is silently
disabled (`can't access tty; job control turned off`).

To rebrand the `uname -a` output (BusyBox hardcodes "GNU/Linux" via Kconfig,
not a source string):

```sh
sed -i 's/CONFIG_UNAME_OSNAME="GNU\/Linux"/CONFIG_UNAME_OSNAME="Busybox\/Linux"/' .config
```

### 4. Root filesystem layout

```
/init                 - initramfs entry point (mounts proc/sys/dev, execs /sbin/init)
/sbin/init            - symlink to busybox (BusyBox init)
/etc/inittab          - sysinit, respawn shell/getty, ctrlaltdel, shutdown
/etc/init.d/rcS        - sysinit script (mounts tmpfs, sets hostname, brings up network)
/etc/passwd /etc/shadow /etc/group /etc/securetty
/etc/profile           - sets PATH for all users
/etc/issue             - pre-login banner
/etc/motd              - post-login banner
```

### 5. Package initramfs and ISO

```sh
cd rootfs
find . | cpio -o -H newc | gzip -9 > ../initramfs.cpio.gz
```

```sh
mkisofs -o netroot-linux.iso \
  -V "NETROOT_LINUX" \
  -b boot/isolinux/isolinux.bin \
  -c boot/isolinux/boot.cat \
  -no-emul-boot -boot-load-size 4 -boot-info-table \
  -R -J \
  iso/
```

---

## Known Issues / Gotchas Encountered During Development

These are documented here because they were non-obvious and cost real
debugging time — future builds (or anyone following a similar path) should
watch for them:

1. **"No working init found" with no error code** → almost always
   `CONFIG_BINFMT_ELF`/`CONFIG_BINFMT_SCRIPT` missing. `tinyconfig` disables
   both.
2. **Same panic, but with `error -8` (ENOEXEC)** → either a 32/64-bit
   mismatch between kernel and userland binaries, or CPU-specific
   instructions (AVX/`vmovdqu`/`vzeroupper`) baked into musl/BusyBox by an
   inherited `-march=native` from the host's Portage `CFLAGS`. Check with
   `objdump -d busybox | grep -iE "vzeroupper|vmovdqu|ymm|zmm"`.
3. **`can't set groups: Function not implemented`** on non-root login →
   `CONFIG_MULTIUSER` disabled.
4. **`can't access tty; job control turned off`** → shell spawned without
   `cttyhack`; fix by using `/sbin/cttyhack /bin/sh` (or a real `getty`) in
   `/etc/inittab` instead of a bare `/bin/sh`.
5. **Boot works in QEMU but not on real hardware / via Ventoy** → Ventoy's
   default boot mode can mishandle non-standard isolinux ISOs; test with a
   direct `dd`-written USB stick before assuming the ISO itself is broken.
6. **Kernel silently "compiles" almost instantly after a config change** →
   incremental builds only recompile affected objects; this is normal, but
   if in doubt, verify with `ls -la arch/x86/boot/bzImage` (timestamp) or
   force a full `make clean` rebuild.
7. **`ping` works but ICMP raw sockets misbehave / permission issues** →
   largely a non-issue when running as root the whole time (which this
   system does by default), but worth knowing this is a systemic
   raw-socket/capability thing, not a NetRoot bug.

---

## Philosophy / Scope

NetRoot Linux intentionally avoids:

- glibc (musl only)
- systemd / OpenRC / sysvinit (BusyBox init only)
- Any package manager (this is not meant to be extended via packages — it's
  a fixed-purpose minimal system)
- UEFI support (legacy BIOS only, for now)
- Dynamic linking in the base system (BusyBox is fully static; dynamic
  binaries *can* be added later by shipping musl's dynamic `libc.so`
  alongside a matching `ld-musl-*.so.1` symlink, but this is opt-in, not
  part of the base system)

The goal is a system that is smaller and more minimal than Tiny Core Linux
while still being a real, usable Linux environment — not a rescue shell or a
toy.

---

## License

NetRoot Linux is distributed under GPL-2.0 (inherited from the Linux kernel
and BusyBox licensing). See individual component licenses in their
respective source trees under `sources/`.

---

*Built and documented — July 2026.*
