# How to install arch linux
## Preperation

* Downloads the latest arch .iso and .sig from [the arch installation page](https://archlinux.org/download/)

* Verify the signature with: 
``` 
pacman-key -v archlinux-version-x86_64.iso.sig 
```
    
* Write to usb media with:
```
dd if=arch.iso of=/dev/sdx bs=4M status=progress oflag=sync
```

## Installation
* Run the following before doing any further steps:
```
loadkeys se-lat6
ping 1.1.1.1 // or 4.4.4.4 to ensure network connectivity
timedatectl set-ntp true
```
### partitioning
* using cdisk partition your disks to match the following

| Disk  |  Size |  Label    |  Purpose               |
|-------|-------|-----------|------------------------|
|  sdx1 | 300M  |  Boot     |  EFI Boot Partition    |
|  sdx2 | ~80%  |  Arch     |  Main System Partition |
|  sdx3 |  RAM  |  SWAP     |  System Swap Memory    |
|  sdy1 |  100% |  blkStore |  System Bulk Storage   |
---
### encryption
* To encrypt and format the System Partition:
```
cryptsetup luksFormat sdx2 --type luks1
cryptsetup open sdx2 cryptroot
mkfs.btrfs -L Arch /dev/mapper/cryptroot
```
* Create the btrfs subvolumes
```
mount /dev/mapper/cryptroot /mnt
btrfs subvol create /mnt/@
btrfs subvol create /mnt/@snapshots
btrfs subvol create /mnt/@home
btrfs subvol create /mnt/@tmp
```
* unmount and create the basic file system
```
umount /mnt
mount -o subvol=@ /dev/mapper/cryptroot /mnt
mkdir /mnt/{home,tmp,.snapshots}

mount -o subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o subvol=@tmp /dev/mapper/cryptroot /mnt/tmp
mount -o subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots

mkdir -p /var/cache/pacman/
btrfs subvol create /var/cache/pacman/pkg
```
### formating
* Create and mount the EFI
```
mkfs.fat -F 32 /dev/sdx1
mkdir /mnt/efi
mount /dev/sdx1 /mnt/efi 
```

* Create the swap
```
mkswap /dev/sdx3
swapon /dev/sdx3
```

* Create the bulk storage
```
mkfs.ext4 -L blkStore /dev/sdy
mkdir /mnt/blkstore
mount /dev/sdy /mnt/blkstore
```

### pacstrap
* to install Arch linux
```
reflector --country Germany --country Sweden --country Norway --country Denmark --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

//todo: create package list
pacstrap /mnt base base-devel linux linux-firmware sudo neovim neofetch dhcpcd

genfstab -U /mnt >> /mnt/etc/fstab
```

## access
* Initialise the system with the following sequesnse
```
arch-chroot /mnt
ln -sf /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
hwclock --systohc
    uncomment en_US.UTF-8 UTF-8 from /etc/locale.gen
locale-gen

echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=\"se-lat6\"" > /etc/vconsole.conf
echo "<hostname>" > /etc/hostname
```

* Create you /etc/hosts with the following
```
127.0.0.1	localhost
::1		    localhost
127.0.1.1	<hostname>.localdomain	<hostname>
```

### boot
* install and setup
```
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
```

* Edit your /etc/default/grub to match the following
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=/dev/sdx2:cryptroot"
GRUB_ENABLE_CRYPTODISK=y
```

* Edit your /etc/mkinitcpio.conf to include the following
```
HOOKS=(base udev autodetect modconf block filesystems keyboard keymap encrypt fsck openswap)
BINARIES=(/usr/bin/btrfs)
```
* Then run the following 
```
mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg
```

### enviorment setup
* set your root password
```
passwd 
```

* create your user
```
useradd -G [ftp,http,wheel,audio,disk,input,storage,video] -m -U <username>
passwd <username>

EDITOR=nvim visudo //enable sudo for wheel users
```

* install and setup network
```
systemctl enable systemd-networkd
systemctl enable systemd-resolved
systemctl enable dhcpcd 
```
# DONE !!!!

Now reboot
```
reboot
```