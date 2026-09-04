# Guardian (guardian64) - woof-CE build configuration

A minimal, hardened, **close-to-airgapped** x86_64 Puppy Linux built from
Ubuntu 24.04 (noble) binary packages.

## Design

* **Base**: Ubuntu 24.04 (noble) binary packages, woof-CE "upup" style.
* **Kernel**: custom minimal 6.12.x x86_64 kernel built with
  `kernel-kit/guardian-build.conf` (see
  `kernel-kit/configs_x86_64/DOTconfig-6.12.2-x86_64-guardian`).
  * `CONFIG_MODULES=y` - drivers live in the `kernel-modules` SFS (zdrv),
    only boot-critical drivers are built into vmlinuz.
  * No NIC/WLAN/Bluetooth drivers, no network filesystems, no sound/media,
    no staging - only the loopback stack remains.
  * Boot-critical things are built in: SATA/NVMe/USB storage, ext4/vfat/
    iso9660/squashfs, AUFS and overlayfs, EFI framebuffer + simpledrm,
    USB HID/keyboard, serial console.
  * Only common desktop GPUs are kept as modules: i915, amdgpu, nouveau,
    vmwgfx, qxl, virtio-gpu, bochs.
  * Hardening: KASLR, SMAP/UMIP, PTI, retpoline, spectre-BHI, strong stack
    protector, FORTIFY, init-on-alloc/free, slab freelist random/hardening,
    enforced module signatures, kernel lockdown LSM
    (`lockdown=integrity` passed at boot via `EXTRA_KERNEL_PARAMS`),
    dmesg restriction, no kexec.
* **No firmware**: `fware=n` in `guardian-build.conf`; the fdrv is empty.
* **No network stack by default**: firewall (`firewall_ng`) is enabled at
  build time, `/etc/hosts` is local-only, no network managers or browsers.

## Build

```
# from the woof-CE checkout
./merge2out woof-distro/x86_64/ubuntu/guardian64
cd ../woof-out_x86_64_x86_64_ubuntu_guardian64

# 1. build the Guardian kernel (non-interactive)
cd kernel-kit
./build.sh guardian-build.conf auto
cd ..

# 2. build the Puppy
./0setup && ./1download && ./2createpackages
./3builddistro release
```

The ISO (`guardian64-1.0.iso`) is UEFI-bootable: it contains a GRUB2 EFI
image (from frugalpup) and an `efilinux` stub; and it is also BIOS-bootable
(isolinux) and usable as a frugal install.
