# NetRoot Linux – Minimal Linux System

NetRoot Linux is a lightweight, bootable Linux system built from scratch.  
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

Download or build the ISO image (`netroot.iso`). You can run it with QEMU:

```sh
qemu-system-x86_64 -cdrom netroot.iso -display sdl
```

For a serial console (output in the terminal):

```sh
qemu-system-x86_64 -cdrom netroot.iso -nographic
```

## After Boot – Network Setup

When the system starts, you can login:
```
login: root, passwd: root \\ login: netroot, passwd: netroot
```

Once you login, you’ll get a shell (`~#`). To enable networking:

```sh
ip link set eth0 up
udhcpc -i eth0 -s /etc/udhcpc.conf
```

Test connectivity with:

```sh
wget -O - http://example.org
```

## License

NetRoot Linux is distributed under the GPL‑2.0 license. See `Sources/README.md` for component details.

---

*July 2026*
