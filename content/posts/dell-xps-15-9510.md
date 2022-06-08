---
title: "Dell Xps 15 9510"
date: 2022-06-08T22:06:32+02:00
draft: false
toc: false
images:
tags:
  - linux
  - arch
  - dell
  - xps
---

# Setup


## Connect 

```sh
iwctl
```

```sh
device list
station <device> scan
station <device> get-networks
station <device connect <SSID>
```
```sh
ping google.com
```

## Date and Time

```sh
timedatectl set-ntp true
timedatectl status
```

Set timezone

```sh
ln -sf /usr/share/zoneinfo/Europe/Brussels /etc/localtime
```

Sync the hardware clock

```sh
date
hwclock --systohc
```

## Disk

Format the disk with gdisk, creating 2 partitions:

* **p1**: 1GB / EFI
* **p2**: Rest / Linux Crypt

### Formatting

Create the EFI Fs

```sh
mkfs.fat -F 32 -n EFI /dev/nvme0n1p1
```

#### Crypt

```sh
cryptsetup luksFormat /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 arch
lsblk
```

#### Btrfs

```sh
mkfs.btrfs /dev/mapper/arch
mount /dev/mapper/arch /mnt
ls /mnt
btrfs  subvolume create /mnt/@
btrfs  subvolume create /mnt/@home
btrfs  subvolume create /mnt/@var
btrfs  subvolume create /mnt/@swap
btrfs  subvolume create /mnt/@pacman
btrfs  subvolume create /mnt/@vms
umount /mnt
```

Mount the subvolumes to their respective places

```sh
mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@ /dev/mapper/arch /mnt
mkdir /mnt/{boot,home,var,swap,var/cache/pacman/pkg,vms,swap}
mkdir -p /mnt/{boot,home,var,swap,vms,swap}
mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@home /dev/mapper/arch /mnt/home
mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@var /dev/mapper/arch /mnt/var
mkdir -p /mnt/var/cache/pacman/pkg
mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@pacman /dev/mapper/arch /mnt/var/cache/pacman/pkg
mount -o noatime,nodatacow,ssd,discard=async,space_cache=v2,subvol=@vms /dev/mapper/arch /mnt/vms
mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@swap /dev/mapper/arch /mnt/swap
mount /dev/nvme0n1p1 /mnt/boot
```

# Install the base system

```sh
pacstrap /mnt base linux linux-firmware git vim btrfs-progs
```

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```
> **Tip**: Now is a good time to backup your history file back onto the USB you've booted with. This prevents you form manually re-entering all the mount commands again in case you do need to boot again from the install media. Just copy the `history.txt` file back to `/root/.zsh_history` and open a new tty with <kbd>Alt+Left</kbd>
> ```sh
>  mkdir /usb
>  mount /dev/sda1 /usb
>  cp .zsh_history /usb/history.txt
>  ```

# Chroot!

```sh
arch-chroot /mnt
```


