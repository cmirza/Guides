# UTM - Arch Linux Install

This guide outlines setting up Arch Linux with UTM on macOS for Apple Silicon, tested on a 2022 MacBook Pro with M2 Pro.

## Download UTM

Download and install UTM.

From UTM's site:
[https://mac.getutm.app](https://mac.getutm.app)

or

From App Store:
[https://apps.apple.com/us/app/utm-virtual-machines/id1538878817](https://apps.apple.com/us/app/utm-virtual-machines/id1538878817)

## Download Archboot

Download latest build of Archboot for aarch64: [https://archboot.net/iso/aarch64/latest/](https://archboot.net/iso/aarch64/latest/)

File name should look like: /archboot-{date}-{time}-{version}-aarch64-ARCH-latest-aarch64.iso

*Example: https<nolink>://archboot.net/iso/aarch64/latest/archboot-2023.09.15-11.59-6.2.10-1-aarch64-ARCH-latest-aarch64.iso*


## UTM Setup
- Run UTM

- Select 'Virtualize'

- Select 'Linux'

- Boot ISO Image -> Browse -> *path to Archboot ISO*

- Memory: 4096 MB (or more if you have it)

- CPU Cores: Default

- Storage: 10 GB (or more if you have it)

- Shared: Continue (Skip)

- Name: Arch Linux

- Edit VM

*optional*

- Download Arch Icon: [https://cdn0.iconfinder.com/data/icons/flat-round-system/512/archlinux-512.png](https://cdn0.iconfinder.com/data/icons/flat-round-system/512/archlinux-512.png)

- Icon -> Custom -> *path to image*

## Load VM

- Run VM

- Will run through automated basic Setup

- At `Enter for login routine`, use Ctrl+C to exit

## Set Console Keyboard Layout
Set keyboard layout 
```
# loadkeys us
```
To see all available keymaps
 ```
 # localectl list-keymaps
 ```

## Verify boot mode
Confirm EFI boot 
```
# cat /sys/firmware/efi/fw_platform_size
```
should return `64`

## Verify network connection
Check network connection
```
# ping 1.1.1.1
```

## Update System Clock
Update system clock
```
# timedatectl status
```

## Partition Disks
List available disks

```
# fdisk -l
``` 

You should see `Disk /dev/vda`

`/dev/zram0` is the Archboot ZRAM environment, you can ignore this one

Now setup partition tables 

```
# fdisk /dev/vda
```

I will setup UEFI with GPT example

Mount point	| Partition	| Partition Type | Size
:--- | :--- | :--- |---:|
`/mnt/boot1` | `/dev/efi_system_partition` | `EFI system partition` | 512MB |
`[SWAP]` | `/dev/swap_partition` | `Linux swap` | 512MB |
`/mnt` | `/dev/root_partition` | `Linux x86-64 root (/)` | 9GB |

Type `g` create to GPT partition

### Create EFI System Partition

Type `n` to add a new partition

Partition number `1` (default)

First sector `2048` (default)

Last sector `+512M`

Type `t` to choose type

Partition number `1` (default)

Set as `uefi`

### Create Linux Swap Partition

Type `n` to add a new partition

Partition number `2` (default)

First sector `1050624` (default)

Last sector `+512M`

Type `t` to choose type

Partition number `2` (default)

Set as `swap`

### Create Root Partition

Type `n` to add a new partition

Partition number `3` (default)

First sector `2099200` (default)

Last sector `20969471` (default)

Type `t` to choose type

Partition number `3` (default)

Set as `linux`

### Save changes

Type `w` to write to disk

Confirm changes

Run `# fdisk -l` and confirm partitions are created

## Format Partitions

### Root

`# mkfs.ext4 /dev/vda3`

### Swap

`# mkswap /dev/vda2`

### Boot

`# mkfs.fat -F 32 /dev/vda1`

## Mount File Systems

### Root

`# mount /dev/vda3 /mnt`

### EFI System

`# mount --mkdir /dev/vda1 /mnt/boot`

### Swap

`# swapon /dev/vda2`

## Installation

### Install essential packages 

```
pacstrap -K /mnt base base-devel linux linux-firmware e2fsprogs dhcpcd networkmanager archlinuxarm-keyring vim neovim man-db man-pages texinfo
```

### Fstab

Generate fastab

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

Confirm no errors in created file

```
# vim /mnt/etc/fstab
```

### Chroot

Change root into new system

```
# arch-chroot /mnt
```

### Time Zone

Set time zone

```
# ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
```

Run hwclock

```
# hwclock --systohc
```

ℹ️ **To list available regions**

```
# ls /usr/share/zoneinfo/
```

then list time zones

```
# ls /usr/share/zoneinfo/{region}
```

### Localization

Open locales 

```
# vim /etc/locale.gen
```

Find locale `en_US`

Uncomment `en_US.UTF-8 UTF-8` and `en_US ISO_8859-1`

Generate locales

```
# locale-gen
```

Create locale.conf 

```
vim /etc/locale.conf
``` 

and add:

```
LANG=en_US.UTF-8
```

Set console keyboard layout

```
# vim /etc/vconsole.conf
```

and add:

```
KEYMAP=us
```

###  Network Configuration

Create hostname file

```
# vim /etc/hostname
```

and add:

```
cmirza-linux
```
Setup hostname

```
# vim /etc/hosts
```

and add

```
127.0.0.1       localhost
::1             localhost
127.0.1.1       cmirza-linux
```

### Initramfs

Recreate initramfs

```
# mkinitcpio -P
```

### Root password

Set root password

```
# passwd
```

### Bootloader

Download bootloader

```
# pacman -S grub efibootmgr
```

Install grub

```
# grub-install --efi-directory=/boot --bootloader-id=GRUB
```

Create grub config

```
# grub-mkconfig -o /boot/grub/grub.cfg
```

### Final Steps

Exit chroot

```
# exit
```

Unmount all partitions

```
# umount -R /mnt
```

Turn off VM 

```
poweroff now
```

Eject install media for Arch VM in UTM

## Configure System

Start VM and login as root

### Network

Start network

```
# systemctl start NetworkManager
```

Enable network at boot

```
# systemctl enable NetworkManager
```

### System

Update the system

```
# pacman -Syu
```

### Install sudo

```
# pacman -S sudo
```

### Add personal user account 

```
# useradd -m -g users -G wheel,storage,power,video,audio cmirza
# passwd cmirza
```

Grant root to your user

`# EDITOR=nvim visudo`

Uncomment the line:

`%wheel ALL=(ALL:ALL) NOPASSWD: ALL`

Now login with personal user account

`# su cmirza`

### Create XDG user directories

```
sudo pacman -S xdg-user-dirs
xdg-user-dirs-update
```

Remove unneeded directories in these files

`$ sudo vim /etc/xdg/user-dirs.defaults`

`$ vim ~/.config/user-dirs.dirs`

`dd` out unwanted directories

Update XDG

`$ xdg-user-dirs-update`

### Instal AUR Package Manger

```
$ sudo pacman -S git
$ mkdir aur
$ cd aur
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
```
Then delete the `aur` directory
```
$ rm -rf aur
```

### Install SPICE Support On Guest

`$ sudo pacman -S spice-vdagent xf86-video-qxl`

### Audio

```
$ sudo pacman -S pulseaudio
$ sudo pacman -S alsa-utils alsa-plugins
$ sudo pacman -S pavucontrol
```

Install PulseAudio Control Applet

```
$ yay -S pa-applet-git
```

### Network

```
$ sudo pacman -S openssh
$ sudo pacman -S iw wpa_supplicant
```

Install NetworkManager Applet

```
$ sudo pacman -S network-manager-applet
```

Enable SSH, DHCP and NetworkManager

```
$ sudo systemctl enable sshd
$ sudo systemctl enable dhcpcd
$ sudo systemctl enable NetworkManager
```

### Bluetooth❓

May not be necessary on UTM or VM

```
$ sudo pacman -S bluez bluez-utils blueman
$ sudo systemctl enable bluetooth
```

### Pacman

Beautify Pacman:

```
sudo nvim /etc/pacman.conf
```

Uncomment `Color` and add `ILoveCandy` underneath. Uncomment `ParallelDownloads = 5`

### Enable SSD Trim

```
sudo systemctl enable fstrim.timer
```

### Enable Time Synchronization

```
$ sudo pacman -S ntp
$ sudo systemctl enable ntpd
```

## GUI Setup

### Xorg

```
$ sudo pacman -S xorg-server xorg-apps xorg-xinit xclip xdotool xorg-drivers
```

### i3

```
$ sudo pacman -S i3
```

Create `.xinitrc` in home directory

```
$ cd
$ vim .xinitrc
```
Add to file

```
exec i3
```

### Compositor

```
$ sudo pacman -S picom
```

### Fonts

```
$ sudo pacman -S noto-fonts noto-fonts-emoji ttf-firacode-nerd
```

### Shell
```
$ sudo pacman -S zsh
```
Change default shell to zsh
```
$ chsh -s $(which zsh)
```

### Terminal
```
$ sudo pacman -S rxvt-unicode alacritty kitty
```

### Switcher
```
$ sudo pcaman -S dmenu rofi
```

### Status Bar
```
$ sudo pacman -S polybar
```

### File Manager
```
$ sudo pacman -S ranger
```
For previews, feh and uberzug (uberzugpp❓)
```
$ sudo pacman -S feh ueberzug
```

### Browser
```
$ sudo pacman -S firefox
```

### Media Player
```
$ sudo pacman -S mpv
```

### PDF Viewer
```
$ sudo pacman -S zathura zathura-pdf-mupdf
```

### Misc Tools
CLI Utilities
```
$ sudo pacman -S tldr fzf tar gzip htop wget curl neofetch
```
- *tldr* - command cheat sheet
- *fzf* - fuzzy finder
- *tar* - tar/untar
- *gzip* - zip/unzip
- *htop* - CLI task manager
- *neofetch* - system information

```
$ sudo pacman -S tree-sitter tree-sitter-cli
```
- *tree-sitter / tree-sitter-cli* - syntax highlighting in Neovim

GUI Utilities
```
sudo pacman -S maim
```
- *maim* - screenshot utility

Development Tools
```
$ sudo pacman -S codespell nodejs npm yarn python python-pip jre-openjdk jdk-openjdk 
```

### Reboot
```
sudo reboot
```

## i3 Setup

Start i3
```
$ startx
```

Select "Win as default modifier"

## Done

Now ready for your dotfiles
