# 🐧 Linux Install Guides

> Personal installation guides for manually setting up Arch Linux, Artix Linux, and NixOS from scratch —
> written and maintained for my own use case.

---

## 📁 Guides

### [Arch Linux](./arch-linux/)

> `btrfs subvolumes` · `LUKS encryption` · `swapfile` · `zram` · `systemd`

A manual Arch Linux installation with full disk encryption via LUKS, and a Btrfs subvolume layout for root. Uses `openresolv` for DNS, `zram-generator` for zram, and `iwd` + `NetworkManager` for networking.

| #   | Chapter                                                          |
| --- | ---------------------------------------------------------------- |
| 01  | [Connection Setup](./arch-linux/01_ConnectionSetup.md)           |
| 02  | [Drive Setup](./arch-linux/02_DriveSetup.md)                     |
| 03  | [Base Install](./arch-linux/03_BaseInstall.md)                   |
| 04  | [Swap & Zram](./arch-linux/04_SwapZram.md)                       |
| 05  | [Timezone & Locale](./arch-linux/05_TZandLocale.md)              |
| 06  | [Hostname & Hosts](./arch-linux/06_Hosts.md)                     |
| 07  | [Users](./arch-linux/07_Users.md)                                |
| 08  | [Services](./arch-linux/08_Services.md)                          |
| 09  | [Network Configuration](./arch-linux/09_NetworkConfiguration.md) |
| 10  | [GRUB & Initramfs](./arch-linux/10_GRUBandInitramfs.md)          |
| 11  | [LUKS Key](./arch-linux/11_LUKSKey.md)                           |

---

### [Artix Linux — dinit](./artix-dinit/)

> `btrfs subvolumes` · `LUKS encryption` · `swapfile` · `zram` · `dinit`

A manual Artix Linux installation using dinit as the init system — a systemd-free alternative based on Arch. Uses `openresolv` for DNS, `zramen` for zram, and `iwd` + `NetworkManager` for networking.

| #   | Chapter                                                           |
| --- | ----------------------------------------------------------------- |
| 01  | [Connection Setup](./artix-dinit/01_ConnectionSetup.md)           |
| 02  | [Drive Setup](./artix-dinit/02_DriveSetup.md)                     |
| 03  | [Base Install](./artix-dinit/03_BaseInstall.md)                   |
| 04  | [Swap & Zram](./artix-dinit/04_SwapZram.md)                       |
| 05  | [Timezone & Locale](./artix-dinit/05_TZandLocale.md)              |
| 06  | [Hostname & Hosts](./artix-dinit/06_Hosts.md)                     |
| 07  | [Users](./artix-dinit/07_Users.md)                                |
| 08  | [Network Configuration](./artix-dinit/08_NetworkConfiguration.md) |
| 09  | [GRUB & Initramfs](./artix-dinit/09_GRUBandInitramfs.md)          |
| 10  | [LUKS Key](./artix-dinit/10_LUKSKey.md)                           |
| 11  | [Post Setup](./artix-dinit/11_PostSetup.md)                       |

---

### [NixOS](./nixos/nixos-full.md)

> `flakes` · `btrfs subvolumes` · `LUKS encryption` · `swapfile` · `zram` · `GRUB`

A single-file NixOS installation guide — imperative one-time setup in the live environment, then everything else declared in a flake. Uses `nixos-install` / `nixos-rebuild` and `iwd` + `NetworkManager` for networking.

---

## ⚙️ Shared Setup

All three guides share the same general philosophy and disk layout strategy:

- **Btrfs subvolumes** instead of separate partitions for `/home`, `/var/log`, `/var/cache`, etc.
- **Unified storage pool** — no need to pre-allocate fixed partition sizes per directory
- **LUKS** for full disk encryption
- **Swapfile on Btrfs** with CoW disabled, alongside compressed zram for in-memory swap
- **Separate `/boot`** partition left unencrypted for GRUB compatibility

---

## ⚠️ Disclaimer

These guides are written **for personal use** and reflect my own setup and preferences.
They are not intended as universal references. Follow at your own risk.
