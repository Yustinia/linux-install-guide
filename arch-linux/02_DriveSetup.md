# Drive Setup

Use `lsblk -f` to list all available disk devices, alongside identifying the target disk.

## Partitioning

> The guide will use `/dev/nvme0n1` as the device.

Use `cfdisk /dev/nvme0n1` to interactively partition them:

| Block            | Type             | Size      | Purpose             |
| ---------------- | ---------------- | --------- | ------------------- |
| `/dev/nvme0n1p1` | EFI System       | 256MiB    | EFI/ESP             |
| `/dev/nvme0n1p2` | Linux filesystem | 512 MiB   | BOOT                |
| `/dev/nvme0n1p3` | Linux filesystem | Remaining | Unified ROOT & HOME |

## Encryption

Encrypt `/dev/nvme0n1p3` with LUKS:

```bash
cryptsetup luksFormat /dev/nvme0n1p3
cryptsetup open /dev/nvme0n1p3 encrypted
```

> You may choose your own name other than **encrypted**

## Formatting

Format & label the partitions:

```bash
mkfs.fat -F32 /dev/vda1
fatlabel /dev/vda1 ESP

mkfs.ext4 -L BOOT /dev/vda2

mkfs.btrfs -L MAIN /dev/mapper/encrypted
```

## Creating Subvolumes

Temporarily mount `/dev/mapper/encrypted` to `/mnt`:

```bash
mount /dev/mapper/encrypted /mnt
```

Then create the subvolumes:

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
btrfs subvolume create /mnt/@var_tmp
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@swap
btrfs subvolume create /mnt/@opt

btrfs subvolume create /mnt/@docker
btrfs subvolume create /mnt/@libvirt
```

> If you use docker & libvirt, it is recommended to create subvolumes for those to avoid bloating the root snapshot

Unmount `/mnt`:

```bash
umount /mnt
```

## Mounting

The following mount options below will be used:

| Option            | Effect                                       |
| ----------------- | -------------------------------------------- |
| `noatime`         | Disables access time tracking (performance)  |
| `compress=zstd:3` | Transparent Zstd compression at level 3      |
| `ssd`             | Enables SSD-optimized I/O heuristics         |
| `discard=async`   | Asynchronous TRIM support for SSDs           |
| `space_cache=v2`  | Uses the v2 free space cache (more reliable) |
| `subvol=<name>`   | Selects which subvolume to mount             |

```bash
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@          /dev/mapper/encrypted /mnt
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@home      /dev/mapper/encrypted /mnt/home
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@var_log   /dev/mapper/encrypted /mnt/var/log
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@var_cache /dev/mapper/encrypted /mnt/var/cache
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@var_tmp   /dev/mapper/encrypted /mnt/var/tmp
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@tmp       /dev/mapper/encrypted /mnt/tmp
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@snapshots /dev/mapper/encrypted /mnt/.snapshots
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@swap      /dev/mapper/encrypted /mnt/swap
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@opt       /dev/mapper/encrypted /mnt/opt

mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=docker    /dev/mapper/encrypted /mnt/var/lib/docker
mount --mkdir -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=libvirt   /dev/mapper/encrypted /mnt/var/lib/libvirt

mount --mkdir /dev/nvme0n1p2 /mnt/boot
mount --mkdir /dev/nvme0n1p1 /mnt/boot/efi
```
