# Guardian — build your own minimal, airgapped x86_64 UEFI Puppy

**Guardian (guardian64)** is a woof-CE build profile for a minimal, hardened,
close-to-airgapped Puppy Linux for **x86_64 UEFI** machines:

* Binary-compatible with **Ubuntu 24.04 (noble)** packages (included in `woof-distro/x86_64/ubuntu/guardian64/`).
* **Custom minimal 6.12.x kernel** built from
  `kernel-kit/guardian-build.conf` + `kernel-kit/configs_x86_64/DOTconfig-6.12.2-x86_64-guardian`:
  modules enabled (drivers live in the kernel-modules SFS), **no NIC / Wi-Fi /
  Bluetooth drivers**, no sound/media/staging, boot-critical drivers built in,
  and hardened (KASLR, SMAP/UMIP, PTI, retpoline, SPECTRE-BHI, FORTIFY,
  enforced module signatures, kernel lockdown LSM, dmesg restriction, no kexec).
* **No firmware** is shipped (`fware=n`; empty fdrv).
* Firewall enabled at build time, local-only `/etc/hosts`, no browsers/network
  apps — nothing but the loopback stack can talk to the network.
* **UEFI-bootable ISO**: GRUB2 EFI image + efilinux stub (and isohybrid BIOS
  boot; frugal-install friendly).

---

## 1. Requirements

* A 64-bit Linux **host** (the build machine).
  * Recommended: a recent Puppy with the devx SFS loaded (the officially
    supported woof host), **or**
  * Any modern Debian/Ubuntu host — add `GITHUB_ACTIONS=1` to the merge2out
    command below (exactly what the official CI does).
* At least **20 GB free disk** (kernel source + build + package downloads).
* `sudo` (root) — woof requires running as root.
* Internet access to `archive.ubuntu.com` / `raw.githubusercontent.com` /
  `www.kernel.org` for packages, PET DBs and the kernel source.

## 2. Get the source

```sh
git clone https://github.com/recursive-ai-dev/woof-CE.git -b arena/01a06a7e-woof-ce
cd woof-CE
```

(or use your own checkout of the `testing` branch including the
`guardian64` profile and the `guardian` kernel files).

## 3. Merge woof (creates the build dir)

```sh
GITHUB_ACTIONS=1 sudo -E bash -c 'yes "" | ./merge2out woof-distro/x86_64/ubuntu/guardian64'
cd ../woof-out_x86_64_x86_64_ubuntu_guardian64
```

## 4. Install host dependencies

Debian/Ubuntu host:

```sh
sudo apt-get update -qq
sudo apt-get install -y --no-install-recommends \
  dc debootstrap librsvg2-bin zstd xml2 syslinux-utils xorriso \
  squashfs-tools git make gcc file bc flex bison bzip2 xz-utils \
  libssl-dev libelf-dev dwarves pkg-config jq

# vercmp (needed by woof/kernel-kit)
[ -f ../local-repositories/vercmp ] || \
  (curl -4 https://raw.githubusercontent.com/puppylinux-woof-CE/initrd_progs/master/pkg/w_apps_static/w_apps/vercmp.c \
     | gcc -x c -o ../local-repositories/vercmp -)
sudo install -m 755 ../local-repositories/vercmp /usr/local/bin/vercmp
sudo install -D -m 644 woof-code/rootfs-skeleton/usr/local/petget/categories.dat /usr/local/petget/categories.dat
sudo ln -sf bash /bin/ash
```

## 5. Build the Guardian kernel (non-interactive, ~30–60 min)

```sh
cd kernel-kit
sudo -E ./build.sh guardian-build.conf auto
cd ..
```

Output: `kernel-kit/output/huge-6.12.2-guardian.tar.bz2` (vmlinuz +
kernel-modules SFS). 3builddistro will pick it up automatically.

> Optional: if you'd rather not compile, comment in `KERNEL_TARBALL_URL=` in
> `_00build.conf` or drop a `huge-*.tar.bz2` into `../woof-out_*/huge_kernel/`.

## 6. Build the Puppy (~1–2 h)

```sh
sudo -E ./0setup          # download package DBs
sudo -E ./1download       # ~1 GB of Ubuntu packages
sudo -E ./2createpackages # cut-down Puppy packages
sudo -E HOME=/root XDG_CONFIG_HOME=/root/.config ./3builddistro release
```

Everything is pre-configured (`_00build.conf`): devx built for petbuilds but
**not** included in the ISO, no bdrv, no Pkg manager, firewall enabled.

## 7. Results

```
woof-output_guardian64-1.0/
├── guardian64-1.0.iso          ← UEFI + BIOS bootable live CD
├── guardian64-1.0.iso.md5.txt / .sha256.txt
├── guardian64_1.0.sfs          ← main Puppy filesystem
├── zdrv_guardian64_1.0.sfs     ← kernel modules
├── fdrv_guardian64_1.0.sfs     ← empty (no firmware)
├── adrv / ydrv
└── devx ...                    ← only if you keep DEVX_IN_ISO=yes
```

## 8. Boot / install

* **Test in a VM** (needs OVMF UEFI firmware):
  ```sh
  sudo apt-get install -y qemu-system-x86 ovmf
  qemu-system-x86_64 -machine q35 -accel kvm -m 2048 \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd \
    -drive if=pflash,format=raw,file=/tmp/guardian_vars.fd \
    -cdrom woof-output_guardian64-1.0/guardian64-1.0.iso
  ```
* **Burn**: `dd if=guardian64-1.0.iso of=/dev/sdX bs=4M conv=fsync` (or use
  a USB writer; the ISO is isohybrid).
* **UEFI USB install**: boot the live ISO, open `BootFlash` → *Create UEFI USB
  (GPT / efilinux)*, or frugal-install to an FAT32 ESP partition.
* **Secure Boot**: the build includes `shim-signed` + `grub-efi-amd64-signed`
  (they land in the devx, which mk_iso uses to produce the Secure Boot EFI
  image). Whether Secure Boot enrolls depends on your machine; disabling
  Secure Boot or enrolling the shim key works either way.

## 9. Security notes

* `EXTRA_KERNEL_PARAMS="lockdown=integrity"` is added to every boot entry in
  `_00build.conf` → verified modules only. If you ever build unsigned modules,
  remove it (or instead set `lockdown=confidentiality`).
* Kernel modules are **signature-enforced** (`MODULE_SIG_FORCE=y`).
* Puppy defaults to root with no password (same as all Puppies). For a
  hardened box, set a password on first boot (`passwd root`).
* The built-in `/etc/hosts` is localhost-only and the firewall is started.
  There are also no network drivers — even if you install network tools,
  there is no hardware path out.

## 10. Customizing

* **Packages**: edit `DISTRO_PKGS_SPECS-ubuntu-noble` — entries are
  `yes|<generic>|<ubuntu pkgs>|...`; ships with 267 `yes` / 45 `no` (network,
  audio/media, printing and bloat removed). The `PETBUILDS` line lists what is
  compiled from source.
* **Kernel**: edit `kernel-kit/configs_x86_64/DOTconfig-6.12.2-x86_64-guardian`
  (or point `DOTconfig_file=` in `guardian-build.conf` at your own config).
* **To add networking** (not recommended for an airgap build): re-enable the
  kernel drivers (`CONFIG_ETHERNET=y`, `CONFIG_WLAN=y`, …), set `fware=f` in
  `guardian-build.conf`, and flip the network packages back to `yes` in the
  package spec.
* **Audio/media**: kernel `SOUND`/`MEDIA_SUPPORT` are off and the related
  packages removed; re-enable both to get them back.
