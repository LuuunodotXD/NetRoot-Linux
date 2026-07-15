# NetLinux – Minimal Linux System

NetLinux is a lightweight, bootable Linux system built from scratch.  
It includes just the essential components to run a functional shell with networking.

## Requirements

- **CPU**: Any x86‑64 (64‑bit) processor – compiled for the generic x86‑64 baseline.
- **RAM**: 32 MB minimum; 64 MB or more is recommended for comfortable networking and multitasking.
- **Storage**: 3MB of storage: the compressed kernel and initramfs together use approximately 2 MB of storage space (the bootable ISO is slightly larger, around 3MB).
- **Boot**: Legacy BIOS (ISOLINUX) – works on any standard PC or virtual machine.

## Components

- **Linux Kernel** 6.6.144  
- **BusyBox** 1.36.1 (provides all user‑space tools)  
- **musl libc** (lightweight C library)
- **isolinux** (bootloader for iso files)

All source code is available in the `sources/` directory and is licensed under GPL‑2.0.

## Quick Start – Boot the ISO

Download or build the ISO image (`netroot.iso`) and run it with QEMU:

```bash
qemu-system-x86_64 -cdrom netlinux.iso -display sdl
```

For a serial console (output in the terminal):

```bash
qemu-system-x86_64 -cdrom netlinux.iso -nographic
```

## After Boot – Network Setup

Once the system starts, you’ll get a shell (`~#`). To enable networking:

```bash
ip link set eth0 up
udhcpc -i eth0 -s /etc/udhcpc.conf
```

Test connectivity with:

```bash
wget -O - http://example.org
```

## License

NetLinux is distributed under the GPL‑2.0 license. See `Sources/README.md` for component details.

---

*July 2026*
