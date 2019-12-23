# notes
- some more help is available:
  - https://linuxconfig.org/install-arch-linux-on-thinkpad-x1-carbon-gen-7-with-encrypted-filesystem-and-uefi
  - https://github.com/ejmg/an-idiots-guide-to-installing-arch-on-a-lenovo-carbon-x1-gen-6
- installs arch with:
  - `LVM` on `LUKS` encryption
  - UEFI
  - `systemd-boot`
  - intel graphics and `ucode` updates
  - a selection of my personal key packages
- guide developed over 2 years as an Arch user
- if I'm doing anything dodgy, questionable or inefficient, please let me know
- always follow the [real installation guide](https://wiki.archlinux.org/index.php/Installation_guide) as a priority since it's constantly updated
- this is confirmed to be working great on:
  - a Thinkpad X1 Carbon 6th Gen with an NVMe SSD
  - a Dell XPS 9360 with an NVMe SSD

# installation process

##### secure boot
Disable `secure boot` on the laptop to be upgraded to Arch.

The method for this varies between laptop but is typically modified in the BIOS.

##### download latest ARCH ISO image
https://www.archlinux.org/download/

##### write to USB
```
dd bs=4M if=/home/ian/downloads/chromium/archlinux-2019.12.01-x86_64.iso of=/dev/sdx status=progress oflag=sync
```

##### wipe drive
```
dd if=/dev/zero of=/dev/nvme0n1 bs=16M status=progress
```

##### check drive
```
badblocks -c 10240 -s -w -t 0 -v /dev/nvme0n1
```

##### initial setup
```
loadkeys uk
ls /sys/firmware/efi/efivars
```

##### internet
```
iw dev
wifi-menu -o wlp58s0
systemctl stop dhcpcd
systemctl start dhcpcd
```

##### date
```
hostnamectl set-hostname archian
timedatectl set-timezone Europe/London
timedatectl set-ntp true
```

##### drives
```
lsblk
cgdisk /dev/nvme0n1
```
- first partition:
  - `512M`
  - `ef00`
  - `efi`
- second partition:
  - (to suggested max)
  - `8300`
  - `main`
```
	mkfs.vfat -F32 /dev/nvme0n1p1
	mkfs.ext4 /dev/nvme0n1p2
```

##### encryption, luks + lvm
```
cryptsetup -v -s 512 luksFormat /dev/nvme0n1p2
crypsetup luksOpen /dev/nvme0n1p2 luks
pvcreate /dev/mapper/luks
vgcreate volume /dev/mapper/luks
lvcreate -L 8G volume -n swap
lvcreate -l +100%FREE volume -n root
pvdisplay
lvdisplay
```

##### prep drives
```
mkswap /dev/mapper/volume-swap
swapon /dev/mapper/volume-swap
mkfs.ext4 /dev/mapper/volume-root
mount /dev/mapper/volume-root /mnt
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
lsblk
```

##### install
```
nano /etc/pacman.d/mirrorlist
```
- change UK mirror to https
- add
	`Server = https://archlinux.uk.mirror.allworldit.com/archlinux/$repo/os/$arch`

```
pacstrap /mnt base base-devel vim dialog wpa_supplicant git fish efibootmgr sudo iw i3 nano linux man-db man-pages texinfo intel-ucode linux-headers linux-lts linux-lts-headers
genfstab -U /mnt >> /mnt/etc/fstab
```
`nano /mnt/etc/fstab`
 - on non-boot, for ssd, change relatime to noatime

##### chroot
```
arch-chroot /mnt
```

##### locale, time, host
```
sed -i 's/^# *\(en-GB.UTF-8\)/\1/' /etc/locale.gen
locale-gen
echo LANG=en_GB.UTF-8 > /etc/locale.conf
export LANG=en_GB.UTF-8
tzselect
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
hwclock --systohc --utc
echo 'KEYMAP=uk' > /etc/vconsole.conf
# echo 'FONT=latarcyrheb-sun32' >> /etc/vconsole.conf
echo 'archhostname' > /etc/hostname
echo '127.0.1.1 archhostname.localdomain archhostname' >> /etc/hosts
hostnamectl set-hostname archhostname
```

##### users
```
passwd
useradd -m -g users -G wheel,storage,power -s /bin/bash ian
passwd ian
echo 'ian ALL=(ALL) ALL' > /etc/sudoers.d/ian
users
```

##### initramfs
```
nano /etc/mkinitcpio.conf
```
- modules:
  add `i915 ext4 nvme intel_agp`
- hooks:
  `base systemd autodetect keyboard sd-vconsole block modconf sd-encrypt sd-lvm2 filesystems fsck`

##### boot
```
bootctl --path=/boot/ install
nano -w /boot/loader/loader.conf
```
```
default arch
timeout 2
editor 0
```

```
blkid -s UUID -o value /dev/mapper/volume-root
cryptsetup luksUUID /dev/nvme0n1p2
nano -w /boot/loader/entries/arch.conf
```
```
title ArchLinux 
linux /vmlinuz-linux 
initrd /initramfs-linux.img
options luks.uuid=UUID luks.name=UUID=luks root=/dev/mapper/volume-root resume=/dev/mapper/volume-swap rw

#options cryptdevice=UUID=<>:lvm:allow-discards resume=/dev/mapper/volume-swap root=/dev/mapper/volume-root rw quiet

###### if boot fucks up
```
cryptsetup luksOpen /dev/nvme0n1p2 luks
mount /dev/mapper/volume-root /mnt
swapon /dev/mapper/volume-swap
mount /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt
```

##### intel graphics
```
nano /etc/modprobe.d/i915.conf
```

```
options i915 enable_rc6=1 enable_fbc=1 semaphores=1 modeset=1 enable_guc_loading=1 enable_guc_submission=1 enable_huc=1 disable_power_well=0 enable_psr=1
```

```
mkdir /etc/X11/xorg.conf.d
nano /etc/X11/xorg.conf.d/20-intel.conf
```
```
Section "Device"
  Identifier  "Intel Graphics"
  Driver      "intel"
	Option      "AccelMethod"
EndSection
```

##### reboot
```
exit
umount -R /mnt
reboot
wifi-menu -o wlp58s0
ip link set wlp58s0 down
netctl start <wifi-ssid-name>
echo 'ForceConnect=yes' >> /etc/netctl/<wifi-ssid-name>
pacman -S intel-ucode
nano /boot/loader/entries/arch.conf
```
```
initrd		/intel-ucode.img
```
```
mkinitcpio -p linux
systemctl stop netctl@<wifi-ssid-name>
systemctl disable netctl@<wifi-ssid-name>
systemctl stop dhcpcd
systemctl disable dhcpcd
```

##### packages
```
nano /etc/pacman.conf
```
- uncomment "Color"

```
sudo pacman -Syu xorg-server xorg-xinit chromium linux-headers xorg xorg-apps dkms ttf-liberation ttf-dejavu xf86-video-intel vulkan-intel libva-intel-driver mesa-libgl libva network-manager-applet networkmanager xf86-input-libinput mesa ntfs-3g fuse python python-virtualenv i3 numlockx noto-fonts ttf-ubuntu-font-family ttf-dejavu ttf-liberation ttf-droid ttf-inconsolata ttf-roboto terminus-font ttf-font-awesome alsa-utils alsa-plugins alsa-lib pavucontrol rxvt-unicode rofi dmidecode systemd-swap neofetch noto-fonts-emoji otf-ipafont ttf-hanazono adobe-source* redshift xorg-xinit mesa xterm xorg-twm xorg-xclock ttf-dejavu ttf-liberation noto-fonts xf86-input-libinput xorg-server xorg-xinit xorg-apps mesa xterm xf86-video-intel lib32-intel-dri lib32-mesa lib32-libgl tmux fwupd utils-linux dmidecode
```

##### internet
```
systemctl start NetworkManager
systemctl enable NetworkManager
nmcli dev wifi connect <wifi-ssid-name> <wi-fi-password> pwd
```

##### yay
```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

##### final steps
```
sudo lspci -s 00:02 -vk
sudo systemctl enable systemd-swap
cp /etc/X11/xinit/xinitrc ~/.xinitrc
nano .xinitrc
```
- remove shit at bottom
```
setxkbmap -layout gb &
numlockx &
xset r rate 200 30 &
exec i3

sudo nano /etc/profile
	add
		if [[ "$(tty)" == '/dev/tty1' ]]; then
    		exec startx
		fi
```
