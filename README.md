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
  - allocation: `512M`
  - type: `ef00`
  - name: `efi`
- second partition:
  - allocation: (to suggested max)
  - type: `8300`
  - name: `main`
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
pacstrap /mnt base base-devel cryptsetup dhcpcd dialog efibootmgr intel-ucode iw linux linux-headers linux-lts linux-lts-headers man-db man-pages nano sudo texinfo vim wpa_supplicant
genfstab -U /mnt >> /mnt/etc/fstab
```
`nano /mnt/etc/fstab`
- on non-boot, for ssd, change relatime to noatime

##### chroot, locale, time, host
```
arch-chroot /mnt
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
- `modules`:
  - add `i915 ext4 nvme intel_agp`
- `hooks`:
  - change line to `base systemd autodetect keyboard sd-vconsole block modconf sd-encrypt sd-lvm2 filesystems fsck`

##### boot
```
bootctl --path=/boot/ install
nano /boot/loader/loader.conf
```
enter: 
```
default arch
timeout 2
editor 0
```

```
# blkid -s UUID -o value /dev/mapper/volume-root
blkid -s UUID -o value /dev/nvme0n1p2
cryptsetup luksUUID /dev/nvme0n1p2
nano /boot/loader/entries/arch.conf
```
enter:
```
title ArchLinux 
linux /vmlinuz-linux 
initrd /initramfs-linux.img
options luks.uuid=<UUID> luks.name=<UUID>=luks root=/dev/mapper/volume-root resume=/dev/mapper/volume-swap rw
#options cryptdevice=UUID=<>:lvm:allow-discards resume=/dev/mapper/volume-swap root=/dev/mapper/volume-root rw quiet
```

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
enter:
```
options i915 enable_rc6=1 enable_fbc=1 semaphores=1 modeset=1 enable_guc_loading=1 enable_guc_submission=1 enable_huc=1 disable_power_well=0 enable_psr=1
```

```
mkdir /etc/X11/xorg.conf.d
nano /etc/X11/xorg.conf.d/20-intel.conf
```
enter:
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
```
```
#pacman -S intel-ucode
#nano /boot/loader/entries/arch.conf
#enter:
  #initrd	/intel-ucode.img
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
sudo pacman -Syu binutils coreutils fish grep gzip htop lib32-intel-dri lib32-libgl lib32-mesa libva libva-intel-driver light mesa mesa-libgl netctl network-manager-applet networkmanager openssh pacman pacman-contrib pciutils rclone sed sshfs systemd-swaptar tmux tree ufw unzip usbutils util-linux utils-linux vulkan-intel wget xf86-input-libinput xf86-video-intel xf86-video-vesa zip
```

##### internet
```
systemctl start NetworkManager
systemctl enable NetworkManager
nmcli dev wifi connect <wifi-ssid-name> <wi-fi-password>
```

##### yay
```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

##### install a bunch of other packages
```
sudo pacman -Syu acpi adobe-source-han-sans-cn-fonts adobe-source-han-sans-hk-fonts adobe-source-han-sans-jp-fonts adobe-source-han-sans-kr-fonts adobe-source-han-sans-otc-fonts adobe-source-han-sans-tw-fonts adobe-source-han-serif-cn-fonts adobe-source-han-serif-jp-fonts adobe-source-han-serif-kr-fonts adobe-source-han-serif-otc-fonts adobe-source-han-serif-tw-fonts adobe-source-sans-pro-fonts adobe-source-serif-pro-fonts alsa-lib alsa-plugins alsa-utils android-tools arandr arc-gtk-theme asciiquarium autoconf automake bash bison bzip2 catimg chafa chromium cmatrix code device-mapper diffutils dkms dmidecode dpkg e2fsprogs fakeroot feh file filesystem findutils flashplugin flex fuse fuse2 fwupd fzf gawk gcc gcc-libs gettext git glibc gnome-keyring gparted gpaste groff gvfs hdparm i3 i3-wm i3blocks i3lock i3status imagemagick inetutils iotop iproute2 iputils jfsutils jre10-openjdk jupyter less libmtp libreoffice-fresh libtool licenses linux-firmware logrotate lsd lvm2 lxappearance m4 make mdadm mpv neko neofetch net-tools nextcloud-client nodejs noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra npm ntfs-3g numlockx okular opera otf-ipafont pamixer patch pavucontrol pcmanfm pepper-flash perl pkgconf playerctl postgresql powertop procps-ng psmisc pulseaudio pulseaudio-alsa pulseaudio-bluetooth python python-virtualenv qalculate-gtk qgis redshift reiserfsprogs rofi rxvt-unicode s-nail scrot seahorse shadow sl slimevolley sublime-text sysfsutils systemd systemd-sysvcompat telegram-desktop terminus-font transmission-gtk ttf-dejavu ttf-droid ttf-font-awesome ttf-hanazono ttf-ibm-plex ttf-inconsolata ttf-joypixels ttf-liberation ttf-roboto ttf-roboto-mono ttf-ubuntu-font-family vlc w3m which whois xclip xfce4-terminal xfsprogs xorg xorg-apps xorg-bdftopcf xorg-docs xorg-font-util xorg-fonts-100dpi xorg-fonts-75dpi xorg-fonts-encodings xorg-iceauth xorg-luit xorg-mkfontscale xorg-server xorg-server-common xorg-server-devel xorg-server-xephyr xorg-server-xnest xorg-server-xvfb xorg-server-xwayland xorg-sessreg xorg-setxkbmap xorg-smproxy xorg-twm xorg-x11perf xorg-xauth xorg-xbacklight xorg-xclock xorg-xcmsdb xorg-xcursorgen xorg-xdpyinfo xorg-xdriinfo xorg-xev xorg-xgamma xorg-xhost xorg-xinit xorg-xinput xorg-xkbcomp xorg-xkbevd xorg-xkbutils xorg-xkill xorg-xlsatoms xorg-xlsclients xorg-xmodmap xorg-xpr xorg-xprop xorg-xrandr xorg-xrdb xorg-xrefresh xorg-xset xorg-xsetroot xorg-xvinfo xorg-xwd xorg-xwininfo xorg-xwud xsel xterm yay zathura

yay -Syu chromium-widevine debtap fisher google-cloud-sdk gotop imagewriter libinput-gestures pipes.sh postman qt4 slack-desktop spotify ttf-roboto-slab ttf-symbola ttf-twemoji vcvrack-bin xorg-server-xdmx
```

##### final steps
```
sudo lspci -s 00:02 -vk
sudo systemctl enable systemd-swap
cp /etc/X11/xinit/xinitrc ~/.xinitrc
nano .xinitrc
```
- remove shit at bottom
- enter: 
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
