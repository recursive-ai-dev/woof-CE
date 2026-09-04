# Guardian (guardian64) - build profile

A minimal, hardened, close-to-airgapped x86_64 Puppy Linux built from
Ubuntu 24.04 (noble) binary packages. See **docs/Guardian-BUILD.md** for the
full build instructions.

## What's here

* `DISTRO_SPECS` - Guardian / 1.0 / prefix `guardian64`
* `DISTRO_PKGS_SPECS-ubuntu-noble` - package selection (267 `yes` / 45 `no`;
  network, audio/media, printing and bloat removed) + trimmed `PETBUILDS`
* `_00build.conf` - UEFI 64-bit only, no firmware, firewall enabled at build,
  local-only `/etc/hosts`, `lockdown=integrity` kernel param, no Pkg manager,
  devx built but not shipped, zstd SFS compression
* `DISTRO_COMPAT_REPOS-ubuntu-noble`, `DISTRO_PET_REPOS` - Ubuntu noble +
  Puppy PET repos
* Kernel (`../kernel-kit/...`): `guardian-build.conf` and
  `configs_x86_64/DOTconfig-6.12.2-x86_64-guardian`
