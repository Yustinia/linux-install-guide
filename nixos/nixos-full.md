# NixOS Installation Guide

> Covers flakes, modular config, btrfs subvolumes, LUKS encryption, zram, and NetworkManager with iwd.

---

## Mental Model

The installation process has two distinct phases. Keep them separate in your head:

| Phase                | What happens                                                           |
| -------------------- | ---------------------------------------------------------------------- |
| **Live environment** | Partition, format, encrypt, mount — imperative shell commands run once |
| **Configuration**    | Everything else — declared in Nix, built by `nixos-install`            |

Anything you do in the live environment is a one-time physical setup. Everything in the configuration phase is reproducible and version-controlled.

---

## Target File Structure

```
/etc/nixos/
├── flake.nix
└── hosts/
    └── nixos/
        ├── default.nix      # system configuration
        └── hardware.nix     # boot, LUKS, filesystems, kernel
```

`hardware.nix` holds everything tied to the physical machine — partition UUIDs, LUKS setup, bootloader, kernel modules. `default.nix` holds everything else. This split matters when you add a second machine later: only `hardware.nix` needs to differ.

---

## 01 — Network

```bash
nmtui
ping -c 3 nixos.org
```

Use `nmtui` to connect wirelessly. Verify connectivity before proceeding.

---

## 02 — Drive Setup

### Identify Your Disk

```bash
lsblk -f
```

This guide assumes `/dev/nvme0n1`. Substitute your actual disk identifier throughout.

### Partition

```bash
cfdisk /dev/nvme0n1
```

| Partition        | Type             | Size      | Purpose        |
| ---------------- | ---------------- | --------- | -------------- |
| `/dev/nvme0n1p1` | EFI System       | 256 MiB   | EFI/ESP        |
| `/dev/nvme0n1p2` | Linux filesystem | 512 MiB   | `/boot`        |
| `/dev/nvme0n1p3` | Linux filesystem | Remaining | LUKS container |

### Encrypt

```bash
cryptsetup luksFormat /dev/nvme0n1p3
cryptsetup open /dev/nvme0n1p3 cryptroot
```

Note the UUID — it goes into `hardware.nix` later:

```bash
blkid -o value -s UUID /dev/nvme0n1p3
```

### Format

```bash
mkfs.fat -F32 /dev/nvme0n1p1
fatlabel /dev/nvme0n1p1 ESP

mkfs.ext4 -L BOOT /dev/nvme0n1p2

mkfs.btrfs -L MAIN /dev/mapper/cryptroot
```

### Create Btrfs Subvolumes

Subvolumes allow granular snapshotting — you can snapshot `@home` independently of `@`, exclude `@var_cache` from backups, and so on. Create them before mounting:

```bash
mount /dev/mapper/cryptroot /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
btrfs subvolume create /mnt/@var_tmp
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@swap
btrfs subvolume create /mnt/@opt

umount /mnt
```

### Mount

```bash
OPTS="noatime,compress=zstd:3,ssd,discard=async,space_cache=v2"

mount --mkdir -o ${OPTS},subvol=@          /dev/mapper/cryptroot /mnt
mount --mkdir -o ${OPTS},subvol=@home      /dev/mapper/cryptroot /mnt/home
mount --mkdir -o ${OPTS},subvol=@var_log   /dev/mapper/cryptroot /mnt/var/log
mount --mkdir -o ${OPTS},subvol=@var_cache /dev/mapper/cryptroot /mnt/var/cache
mount --mkdir -o ${OPTS},subvol=@var_tmp   /dev/mapper/cryptroot /mnt/var/tmp
mount --mkdir -o ${OPTS},subvol=@tmp       /dev/mapper/cryptroot /mnt/tmp
mount --mkdir -o ${OPTS},subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount --mkdir -o ${OPTS},subvol=@swap      /dev/mapper/cryptroot /mnt/swap
mount --mkdir -o ${OPTS},subvol=@opt       /dev/mapper/cryptroot /mnt/opt

mount --mkdir /dev/nvme0n1p2 /mnt/boot
mount --mkdir /dev/nvme0n1p1 /mnt/boot/efi
```

**Note on mount options and `@swap`:** Btrfs mount options apply at the filesystem level, not per-subvolume. `compress=zstd:3` technically applies to `@swap` as well, but is a no-op there — the swapfile is created with CoW disabled via `chattr +C`, and Btrfs does not compress NoCoW files.

---

## 03 — Pre-Install Preparation

These steps must happen before running `nixos-install`.

### Generate Base Config

```bash
nixos-generate-config --root /mnt
```

This produces two files:

- `hardware-configuration.nix` — auto-detected hardware: kernel modules, CPU microcode, filesystems
- `configuration.nix` — a generic starting config with a module skeleton

The workflow is:

1. Move both files to your host directory, renaming them:

```bash
mv /mnt/etc/nixos/hardware-configuration.nix /mnt/etc/nixos/hosts/nixos/hardware.nix
mv /mnt/etc/nixos/configuration.nix /mnt/etc/nixos/hosts/nixos/default.nix
```

2. In `hardware.nix` — do not remove the auto-generated content. Add your LUKS, bootloader, and kernel declarations on top of what is already there. Those values represent what was detected for your specific hardware.

3. In `default.nix` — strip all generated option declarations and comments, leaving only the module skeleton:

```nix
{ config, lib, pkgs, ... }:
{
  imports = [
    ./hardware.nix
  ];

}
```

Then fill it in with your system configuration from here.

If you ever need to re-run `nixos-generate-config`, use `--no-filesystems` to avoid overwriting your filesystem declarations:

```bash
nixos-generate-config --root /mnt --no-filesystems
```

### Create the Swapfile

The swapfile must exist on disk before `nixos-install` runs — NixOS will not create it:

```bash
chattr +C /mnt/swap
btrfs filesystem mkswapfile --size 4G --uuid clear /mnt/swap/swapfile
```

`chattr +C` disables Copy-on-Write on the `/swap` directory. The swapfile inherits this attribute, which is required for Btrfs swapfiles to function correctly.

### Generate the LUKS Keyfile

The keyfile allows the initramfs to unlock the LUKS container automatically, so you don't need to enter a passphrase at boot via GRUB:

```bash
dd if=/dev/urandom of=/mnt/etc/cryptroot.key bs=64 count=1
chmod 400 /mnt/etc/cryptroot.key
cryptsetup luksAddKey /dev/nvme0n1p3 /mnt/etc/cryptroot.key
```

---

## 04 — Configuration

### `flake.nix`

```nix
{
  description = "NixOS configuration";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  };

  outputs = { self, nixpkgs, ... }@inputs: {
    nixosConfigurations.nixos = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./hosts/nixos/default.nix
      ];
    };
  };
}
```

After running `nixos-install`, a `flake.lock` file will be generated. Commit it to version control — it pins your exact nixpkgs revision and ensures reproducible rebuilds.

### `hosts/nixos/hardware.nix`

This file starts as the moved and renamed `hardware-configuration.nix`. Add the following declarations into it — do not remove the auto-generated content. Replace `YOUR-UUID-HERE` with the UUID from step 02.

```nix
{ pkgs, ... }:
{
  # ... (auto-generated content — do not remove)

  # Kernel — add or replace the auto-generated kernelPackages if desired
  boot.kernelPackages = pkgs.linuxPackages_zen;

  # CPU microcode — must match your CPU
  # hardware.cpu.amd.updateMicrocode = true;
  # hardware.cpu.intel.updateMicrocode = true;

  # Firmware
  # hardware.enableRedistributableFirmware = true;
  # hardware.enableAllFirmware = true;

  # LUKS
  # The keyFile is copied into the initramfs at build time via boot.initrd.secrets,
  # then used to unlock the container at boot. Both declarations are required.
  boot.initrd.luks.devices."cryptroot" = {
    device = "/dev/disk/by-uuid/YOUR-UUID-HERE";
    keyFile = "/etc/cryptroot.key";
    allowDiscards = true;
  };

  boot.initrd.secrets = {
    "/etc/cryptroot.key" = "/etc/cryptroot.key";
  };

  # Bootloader
  # Set efiInstallAsRemovable = true if your motherboard does not reliably
  # persist NVRAM boot entries (common on some ASUS and older UEFI firmware).
  boot.loader.grub = {
    enable = true;
    device = "nodev";
    efiSupport = true;
    efiInstallAsRemovable = false;
  };
  boot.loader.efi.efiSysMountPoint = "/boot/efi";
  boot.loader.efi.canTouchEfiVariables = true;

  # Filesystems
  # Filesystems — replaces the auto-generated fileSystems block with labeled
  # device references and explicit Btrfs subvolume options.
  fileSystems =
    let
      btrfsOpts = [ "noatime" "compress=zstd:3" "ssd" "discard=async" "space_cache=v2" ];
      btrfsDev  = "/dev/disk/by-label/MAIN";
    in {
      "/" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@" ];
      };
      "/home" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@home" ];
      };
      "/var/log" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@var_log" ];
      };
      "/var/cache" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@var_cache" ];
      };
      "/var/tmp" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@var_tmp" ];
      };
      "/tmp" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@tmp" ];
      };
      "/.snapshots" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@snapshots" ];
      };
      "/swap" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@swap" ];
      };
      "/opt" = {
        device = btrfsDev;
        fsType = "btrfs";
        options = btrfsOpts ++ [ "subvol=@opt" ];
      };
      "/boot" = {
        device = "/dev/disk/by-label/BOOT";
        fsType = "ext4";
      };
      "/boot/efi" = {
        device = "/dev/disk/by-label/ESP";
        fsType = "vfat";
      };
    };
}
```

### `hosts/nixos/default.nix`

This file starts as the moved and stripped `configuration.nix`. The following shows what it should look like once filled in:

```nix
{ config, lib, pkgs, ... }:
{
  imports = [
    ./hardware.nix
  ];

  # Required to use flakes and the `nix` CLI after first boot.
  # Without this, nixos-rebuild switch will fail.
  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  # Timezone & Locale
  time.timeZone = "Continent/City";
  i18n.defaultLocale = "en_US.UTF-8";
  console.keyMap = "us";

  # Network
  networking.hostName = "nixos";
  networking.hosts = {
    "127.0.1.1" = [ "nixos.localdomain" "nixos" ];
  };
  networking.networkmanager.enable = true;
  networking.networkmanager.wifi.backend = "iwd";
  networking.nameservers = [
    "94.140.14.14"   # AdGuard primary
    "94.140.15.15"   # AdGuard secondary
    "1.1.1.1"        # Cloudflare fallback
    "1.0.0.1"
  ];

  # Users
  users.users.username = {
    isNormalUser = true;
    shell = pkgs.bash;
    extraGroups = [ "wheel" "networkmanager" ];
  };
  security.sudo.wheelNeedsPassword = true;

  # Swap & Zram
  swapDevices = [
    { device = "/swap/swapfile"; }
  ];
  zramSwap = {
    enable = true;
    algorithm = "zstd";
    memoryPercent = 50;
  };
  # Userspace OOM killer — prevents full system lockups under memory pressure
  systemd.oomd.enable = true;

  # Packages
  environment.systemPackages = with pkgs; [
    vim
    btrfs-progs
  ];

  # Do not change this after the initial install. This value pins stateful
  # NixOS behavior to the version used during installation. It is not how
  # you track or perform upgrades.
  system.stateVersion = "25.05";
}
```

---

## 05 — Install

Run these in order. Steps 1 and 2 were already completed in section 03 — listed here for reference.

```bash
# 1. Swapfile — already created in section 03
# 2. LUKS keyfile — already created in section 03

# 3. Run the installer
#    If prompted for a root password, set one or press enter to skip.
nixos-install --flake /mnt/etc/nixos#nixos

# 4. Set the user password
#    nixos-enter is NixOS's dedicated chroot tool. It activates the installed
#    system's profile correctly, which a bare chroot does not.
nixos-enter --root /mnt -c 'passwd username'

# 5. Reboot
reboot
```

---

## Post-Install

On first boot, verify flakes are working:

```bash
nixos-rebuild switch --flake /etc/nixos#nixos
```

If this fails with an error about experimental features, confirm `nix.settings.experimental-features` is present in `default.nix` and was included during install.
