# Arch Linux on Asus ROG Zephyrus G14 (G401II)
My own notes installing Arch Linux with btrfs, disc encryption, auto-snapshots, no-noise fan-curves on my Asus ROG Zephyrus G14

![desk](desk_v3.png)

[TOC]


## Basic Install

### Prepare and Booting ISO

Boot into BIOS and change your Harwareclock to UTC. Now booting Arch Linux, using a prepared Usbstick

Change keyboard layout with  `loadkeys de-latin1-nodeadkeys`

### Networking

For Network i use wireless, if you need wired please check the [Arch WiKi](https://wiki.archlinux.org/index.php/Network_configuration).

Launch `iwctl` and connect to your AP `station wlan0 connect YOURSSID`
Type `exit` to leave.

Update System clock with `timedatectl set-ntp true`

### Format Disk

* My Disk is `nvme0n1`, check with `lsblk`
* Format Disk using `gdisk /dev/nvme0n1` with this simple layout:

	* `o` for new partition table
	* `n,1,<ENTER>,+512M,ef00` for EFI Boot
	* `n,2,<ENTER>,<ENTER>,8300` for the linux partition
	* `w` to save layout


Format the EFI Partition

`mkfs.vfat -F 32 -n EFI /dev/nvme0n1p1`

### Create encrypted filesystem

```bash
cryptsetup luksFormat /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 luks
```

### Create and Mount btrfs Subvolumes

`mkfs.btrfs -f -L ROOTFS /dev/mapper/luks` btrfs filesystem for root partition


Mount Partitions und create Subvol for btrfs. I dont want home, etc in my snapshots, so create subvol for them.

* `mount -t btrfs LABEL=ROOTFS /mnt` Mount root filesystem to /mnt
* `btrfs sub create /mnt/@`
* `btrfs sub create /mnt/@home`
* `btrfs sub create /mnt/@snapshots`
* `btrfs sub create /mnt/@swap`

### Create a btrfs swapfile and remount subvols

```bash
truncate -s 0 /mnt/@swap/swapfile
chattr +C /mnt/@swap/swapfile
btrfs property set /mnt/@swap/swapfile compression none
fallocate -l 16G /mnt/@swap/swapfile
chmod 600 /mnt/@swap/swapfile
mkswap /mnt/@swap/swapfile
mkdir /mnt/@/swap
```

Just unmount with `umount /mnt/` and remount with subvolumes

```bash
mount -o noatime,compress=zstd,commit=120,subvol=@ /dev/mapper/luks /mnt
mkdir -p /mnt/boot
mkdir -p /mnt/home
mkdir -p /mnt/.snapshots
mkdir -p /mnt/btrfs

mount -o noatime,compress=zstd,commit=120,subvol=@home /dev/mapper/luks /mnt/home/
mount -o noatime,compress=zstd,commit=120,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots/
mount -o noatime,commit=120,subvol=@swap /dev/mapper/luks /mnt/swap/

mount /dev/nvme0n1p1 /mnt/boot/

mount -o noatime,compress=zstd,commit=120,subvolid=5 /dev/mapper/luks /mnt/btrfs/
```


Check mountmoints with `df -Th`

### Install the system using pacstrap

```bash
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs nano vi networkmanager amd-ucode
```

After this, generate the filesystem table using
`genfstab -Lp /mnt >> /mnt/etc/fstab`

Add swapfile
`echo "/swap/swapfile none swap defaults 0 0" >> /mnt/etc/fstab `

### Chroot into the new system and change language settings

```bash
arch-chroot /mnt
echo {MYHOSTNAME} > /etc/hostname
echo LANG=de_DE.UTF-8 > /etc/locale.conf
echo LANGUAGE=de_DE >> /etc/locale.conf
echo KEYMAP=de-latin1-nodeadkeys > /etc/vconsole.conf
echo FONT=lat9w-16 >> /etc/vconsole.conf
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
```

Modify `nano /etc/hosts` with these entries. For static IPs, remove 127.0.1.1

```bash
127.0.0.1		localhost
::1				localhost
127.0.1.1		{MYHOSTNAME}.localdomain	{MYHOSTNAME}
```

`nano /etc/locale.gen` to uncomment the following lines

```bash
de_DE.UTF-8 UTF-8
de_DE ISO-8859-1
de_DE@euro ISO-8859-15
en_US.UTF-8
```
Execute `locale-gen` to create the locales now

Add a password for root using `passwd root`


### Add btrfs and encrypt to Initramfs

`nano /etc/mkinitcpio.conf` and add `encrypt btrfs` to hooks between block/filesystems

`HOOKS="base udev autodetect modconf block encrypt btrfs filesystems keyboard fsck `

Also include `amdgpu` in the MODULES section

create Initramfs using `mkinitcpio -p linux`

### Install Systemd Bootloader

`bootctl --path=/boot install` installs bootloader

`nano /boot/loader/loader.conf` delete anything and add these few lines and save

```bash
default	arch.conf
timeout	3
editor	0
```

` nano /boot/loader/entries/arch.conf` with these lines and save.

```bash
title	Arch Linux
linux	/vmlinuz-linux
initrd	/amd-ucode.img
initrd	/initramfs-linux.img
```

copy boot-options with
` echo "options	cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):luks root=/dev/mapper/luks rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf`

### Set nvidia-nouveau onto blacklist

using `nano /etc/modprobe.d/blacklist-nvidia-nouveau.conf` with these lines

```bash
	blacklist nouveau
	options nouveau modeset=0
```

### Leave Chroot and Reboot

Type `exit` to exit chroot

`umount -R /mnt/` to unmount all volumes

Now its time to `reboot` into the new system!

## Finetuning after first Reboot

### Enable Networkmanager

Configure WiFi Connection.

```bash
systemctl enable --now NetworkManager
nmcli device wifi connect "{YOURSSID}" password "{SSIDPASSWORD}"
```

### Enable NTP Timeservice

```bash
systemctl enable --now systemd-timesyncd.service
```

(You may look at `/etc/systemd/timesyncd.conf` for default values and change if necessary)

### Create a new user

First create a new local user and point it to bash

```bash
useradd -m -g users -G wheel,lp,power,audio -s /bin/bash {MYUSERNAME}
passwd {MYUSERNAME}
```

Execute `visudo` and uncomment `%wheel ALL=(ALL) ALL`

Now `exit` and relogin with the new {MYUSERNAME}


Install some Deamons before we reboot

```bash
sudo pacman -Sy acpid dbus
sudo systemctl enable acpid
```

## Setup automatic Snapshots for Pacman

The goal is to have automatic snapshots each time i made changes with pacman. The hook creates a snapshot
to ".snapshots\STABLE". So if something goes wrong i can boot from this snapshot and rollback my system.


### Create the STABLE snapshot and modify Bootloader

First i create the snapshot and changes by hand to test if anythink is working. After that it will be done automaticly by our hook and script.

```bash
sudo -i
btrfs sub snap / /.snapshots/STABLE
cp /boot/vmlinuz-linux /boot/vmlinuz-linux-stable
cp /boot/amd-ucode.img /boot/amd-ucode-stable.img
cp /boot/initramfs-linux.img /boot/initramfs-linux-stable.img
cp /boot/loader/entries/arch.conf /boot/loader/entries/stable.conf
```

Edit `/boot/loader/entries/stable.conf` to boot from STABLE snapshot

```bash
title   Arch Linux Stable
linux   /vmlinuz-linux-stable
initrd  /amd-ucode-stable.img
initrd  /initramfs-linux-stable.img
options ... rootflags=subvol=@snapshots/STABLE rw
```

Now edit the `/.snapshots/STABLE/etc/fstab` to change the root to the new snapshot/STABLE

ˋˋˋ
...
LABEL=ROOTFS  /  btrfs  rw,noatime,.....subvol=@snapshots/STABLE
...
ˋˋˋ

reboot and test if you can boot from the stable snapshot.

### Script for auto-snapshots

Copy the script from my Repo to

`/usr/bin/autosnap` and make it executable with `chmod +x /usr/bin/autosnap`

### Hook for pacman

Copy the script from my Repo to

`/etc/pacman.d/hooks/00-autosnap.hook`

Now each time pacman executes, it launches the `autosnap`script which takes a snapshot from the current system.



## Install Desktop Environment

### Get X.Org and Xfce4

Install xorg and xfce4 packages

```bash
sudo pacman -Sy xorg xfce4 xfce4-goodies xf86-input-synaptics gvfs xdg-user-dirs ttf-dejavu pulseaudio network-manager-applet firefox-i18n-de

sudo localectl set-x11-keymap de pc105 deadgraveacute
xdg-user-dirs-update
```

LightDM Loginmanager

```bash
sudo pacman -S lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings
sudo systemctl enable lightdm
```

Reboot and login to your new Desktop. For Theming i followed this realy nice howto from [Linux Scoop channel](https://www.youtube.com/watch?v=X3siZNJN3ec).


### Oh-My-ZSH

I like to use oh-my-zsh with Powerlevel10K theme

```bash
sudo pacman -Sy git git-lfs curl wget zsh zsh-completions
chsh -s /bin/zsh # (relogin to activate)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
cd .fonts
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
fc-cache -v
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Set `ZSH_THEME="powerlevel10k/powerlevel10k"` in `~/.zshrc`

### Setup Plymouth for nice Password Prompt during Boot

Plymouth is in [AUR](https://aur.archlinux.org) , just clone the Repo and make the Package (i create a subfolder AUR within my homefolder)

```bash
cd AUR
git clone https://aur.archlinux.org/plymouth-git.git
makepkg -is
```

Now modify the Hooks for the Initramfs, Plymouth must be right after "base udev". Delete encrypt hook, it will be replaced by plymouth-encrypt

```bash
HOOKS="base udev plymouth plymouth-encrypt autodetect modconf block btrfs filesystems keyboard fsck
```

Run `mkinitcpio -p linux`

Add some kernel-parameters to make boot smooth. Edit `/boot/loader/entries/arch.conf` and append to options

```bash
...rootflags=subvol=@ quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 rw
```

Optional: replace LightDM with the plymouth variant

```bash
sudo systemctl disable lightdm
sudo systemctl enable lightdm-plymouth
```

For Plymouth Theming and Options, check [Plymouth on Arch Wiki](https://wiki.archlinux.org/title/plymouth)

For XFCE4 Theming you could check this nice [Youtube Video from Linux Scoop](https://www.youtube.com/watch?v=X3siZNJN3ec)


## Nvidia Driver

### Install latest nvidia driver

```bash
sudo pacman -Sy nvidia nvidia-dkms nvidia-settings acpi_call linux-headers
```

Reboot after install.

## Tweaks

### Add Custom Repo from [Luke Jones](https://asus-linux.org/) and install some tools

#### Add G14 Repository

```bash
sudo bash -c "echo -e '\r[g14]\nSigLevel = DatabaseNever Optional TrustAll\nServer = https://arch.asus-linux.org\n' >> /etc/pacman.conf"

sudo pacman -Sy asusctl supergfxctl
sudo systemctl enable --now power-profiles-daemon.service
sudo systemctl enable --now supergfxd
```

#### Install custom Kernel and change bootloader

```bash
sudo pacman -Sy linux-g14 linux-g14-headers
sudo sed -i 's/vmlinuz-linux/vmlinuz-linux-g14/' /boot/loader/entries/arch.conf
sudo sed -i 's/initramfs-linux/initramfs-linux-g14/' /boot/loader/entries/arch.conf
```

Reboot now!

After reboot you can check available features with `asusctl -s` . Fan curves, profiles should be possible with custom kernel.

#### Modify profiles for asusctl

You can use my file from `etc/asusd/profiles.conf`. But its better to delete the existing file, after reboot its regenerated with default and read-out values, The file is self -descriptive.

#### Activate DBUS Messaging for the new asus deamon

```bash
systemctl --user enable asus-notify
systemctl --user start asus-notify
```



For more fine-tuning read the [asusctl Manual](https://asus-linux.org/asusctl/)

### Setup Bluetooth

Install required bluetooth modules and add your user to the group `lp`
```bash
sudo pacman -Sy bluez bluez-utils blueman
sudo systemctl enable --now bluethooth.service
```
