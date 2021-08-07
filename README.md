# notes
- this process installs arch with:
  - `UEFI` boot mode
  - `LVM` on `LUKS` encryption via `dm-crypt`
  - `systemd-boot`
  - intel/amd graphics with `ucode` updates
  - a selection of packages
- guide developed over 2 years as an Arch user, beware
- if I'm doing anything dodgy, questionable or inefficient, please let me know
- always follow the [real installation guide](https://wiki.archlinux.org/index.php/Installation_guide) as a priority since it's constantly updated
- something very similar to this process has been tested and considerably used on:
  - a Thinkpad X1 Carbon 6th Gen with an NVMe SSD
  - a Dell XPS 9360 with an NVMe SSD
  - a custom built PC (where I used amdgpu video driver instead) with:
      - AMD Ryzen 5 3600 CPU
      - RX 580 GPU
      - 2 x 8 GB Corsair Vengeance RGB Pro memory
      - MSI B450M Mortar Max motherboard
      - Seasonic Focus Plus Gold 550W PSU
      - Dual booting Windows 10 on a seperate NVME SSD

# installation process

##### backup current config to private `dotfiles` repo

```
mkdir -p ~/dotfiles/home/usr/share/
cp -r /usr/share/applications/ ~/dotfiles/home/usr/share/
# cp /usr/share/X11/xorg.conf.d/10-amdgpu.conf ~/dotfiles/home/
cp /etc/X11/xorg.conf.d/20-amdgpu.conf ~/dotfiles/home/
cp /etc/X11/xorg.conf.d/10-monitor.conf ~/dotfiles/home/

mkdir -p ~/dotfiles/home/joplin-desktop
cp ~/.config/joplin-desktop/settings.json ~/dotfiles/home/joplin-desktop/

cp ~/.local/share/fish/fish_history ~/dotfiles/home/
cp ~/.xinitrc ~/dotfiles/home/dotxinitrc
cp ~/.gtk-bookmarks ~/dotfiles/home/dotgtk-bookmarks
cp ~/.Xresources ~/dotfiles/home/dotXresources
cp -r ~/.screenlayout/ ~/dotfiles/home/dotscreenlayout
# navigate to vscode folder location
cp -r .vscode ~/dotfiles/home/dotvscode

sudo cp /etc/hosts ~/dotfiles/home/
sudo cp /boot/loader/entries/arch.conf ~/dotfiles/home/
cp /etc/asound.conf ~/dotfiles/home/
cp /etc/vconsole.conf ~/dotfiles/home/
cp /etc/mkinitcpio.conf ~/dotfiles/home/
cp /etc/pacman.conf ~/dotfiles/home/

cp -r ~/.config/i3/ ~/dotfiles/home/
cp -r ~/.config/i3status-rust/ ~/dotfiles/home/
cp -r ~/.config/xfce4/ ~/dotfiles/home/
cp -r ~/.config/fish/ ~/dotfiles/home/
cp -r ~/.config/fisher/ ~/dotfiles/home/
cp -r ~/.config/nano/ ~/dotfiles/home/
cp -r ~/.config/htop/ ~/dotfiles/home/
cp -r ~/.config/picom/ ~/dotfiles/home/
cp -r ~/.config/pcmanfm/ ~/dotfiles/home/
cp -r ~/.config/ckb-next/ ~/dotfiles/home/
cp -r ~/.config/rofi/ ~/dotfiles/home/
cp ~/.config/redshift.conf ~/dotfiles/home/

pacman -Qqem > ~/dotfiles/home/pacman-Qqem.txt
pacman -Qqen > ~/dotfiles/home/pacman-Qqen.txt
```


##### secure boot
Disable `secure boot` on the laptop to be upgraded to Arch.

The method for this varies between laptop but is typically modified in the BIOS.

##### download latest ARCH ISO image
https://www.archlinux.org/download/

##### write to USB
```
sudo dd bs=4M if=/home/ian/downloads/torrent/archlinux-2021.06.01-x86_64.iso of=/dev/sde status=progress oflag=sync
```

##### wipe drive
```
dd if=/dev/zero of=/dev/nvme0n1 bs=16M status=progress
```

##### check drive
```
badblocks -c 10240 -s -w -t 0 -v /dev/nvme0n1
```

##### boot arch iso

- reboot into arch iso

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
- delete all existing partitions
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
cryptsetup luksOpen /dev/nvme0n1p2 luks
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
pacstrap /mnt base base-devel cryptsetup dhcpcd dialog efibootmgr intel-ucode iw linux linux-headers lvm2 man-db man-pages nano sudo texinfo vim wpa_supplicant wifi-menu vulkan-intel xf86-video-intel linux-firmware xf86-video-vesa
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
```

##### initramfs
```
nano /etc/mkinitcpio.conf
```
- `modules`:
  - for intel, add `i915 ext4 nvme intel_agp`
  - for amd, add `amdgpu ext4 nvme`
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
timeout 0
editor 0
```
get drive UUID for boot loader conf
```
blkid -s UUID -o value /dev/nvme0n1p2
cryptsetup luksUUID /dev/nvme0n1p2
```
```
nano /boot/loader/entries/arch.conf
```
enter:
```
title ArchLinux 
linux /vmlinuz-linux 
initrd /initramfs-linux.img
options luks.uuid=<UUID> luks.name=<UUID>=luks root=/dev/mapper/volume-root resume=/dev/mapper/volume-swap rw
```

##### if boot fucks up
```
cryptsetup luksOpen /dev/nvme0n1p2 luks
mount /dev/mapper/volume-root /mnt
swapon /dev/mapper/volume-swap
mount /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt
```

##### for intel graphics
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
mkinitcpio -p linux
exit
umount -R /mnt
reboot
```
##### wifi 
```
wifi-menu -o wlp58s0
ip link set wlp58s0 down
netctl start <wifi-ssid-name>
echo 'ForceConnect=yes' >> /etc/netctl/<wifi-ssid-name>
systemctl stop netctl@<wifi-ssid-name>
systemctl disable netctl@<wifi-ssid-name>
systemctl stop dhcpcd
systemctl disable dhcpcd
```
##### for intel
```
pacman -S intel-ucode
nano /boot/loader/entries/arch.conf
```
enter:
`initrd	/intel-ucode.img`

```
mkinitcpio -p linux
```

##### packages
```
sudo pacman -Syu acpi adobe-source-han-sans-cn-fonts adobe-source-han-sans-hk-fonts adobe-source-han-sans-jp-fonts adobe-source-han-sans-kr-fonts adobe-source-han-sans-otc-fonts adobe-source-han-sans-tw-fonts adobe-source-han-serif-cn-fonts adobe-source-han-serif-jp-fonts adobe-source-han-serif-kr-fonts adobe-source-han-serif-otc-fonts adobe-source-han-serif-tw-fonts adobe-source-sans-pro-fonts adobe-source-serif-pro-fonts alsa-lib alsa-plugins alsa-utils amd-ucode android-tools arandr asciiquarium autoconf automake bash binutils cmatrix code coreutils device-mapper diffutils dmidecode e2fsprogs fakeroot feh file filesystem findutils firefox fish fuse fuse2 fwupd fzf gawk gcc gcc-libs github-cli gnome-keyring gparted grep gvfs gzip htop i3 i3-wm i3status-rust imagemagick inetutils iotop iproute2 iputils less lib32-libva-mesa-driver lib32-mesa-vdpau lib32-vulkan-radeon libreoffice-fresh libtool libva-mesa-driver licenses lm_sensors logrotate lsd lvm2 lxappearance make mesa mesa-vdpau mpv neofetch net-tools nitrogen ntfs-3g numlockx openssh opera opera-ffmpeg-codecs otf-ipafont pacman pacman-contrib pciutils pcmanfm playerctl pulseaudio pulseaudio-alsa python python-virtualenv qalculate-gtk radeontop rclone redshift reiserfsprogs rofi scrot seahorse sed sshfs syncthing systemd systemd-swap systemd-sysvcompat telegram-desktop terminus-font thunderbird tmux transmission-gtk tree ttf-dejavu ttf-droid ttf-font-awesome ttf-hanazono ttf-ibm-plex ttf-inconsolata ttf-joypixels ttf-liberation ttf-roboto ttf-roboto-mono ttf-ubuntu-font-family ufw unzip usbutils util-linux vdpauinfo vivaldi vlc vulkan-radeon wget which whois xdg-user-dirs xf86-video-amdgpu xfce4-terminal xfce4-settings xorg xorg-apps xorg-xset xclip zerotier-one zip
```

##### wifi
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

##### install AUR packages
```
yay -Syu aur/adwaita-dark aur/archdroid-icon-theme aur/chromium-widevine aur/awesome-terminal-fonts-patched aur/birdtray aur/dislocker aur/exodus aur/google-cloud-sdk aur/joplin-desktop aur/msi-rgb aur/nerd-fonts-roboto-mono aur/protonmail-bridge aur/slack-desktop aur/spotify aur/steam-fonts aur/ttf-ms-fonts aur/vivaldi-widevine aur/whatsapp-nativefier aur/ttf-roboto-slab aur/ttf-twemoji aur/ttf-symbola aur/code-marketplace
```

##### add sublime text repo and install

```
curl -O https://download.sublimetext.com/sublimehq-pub.gpg && sudo pacman-key --add sublimehq-pub.gpg && sudo pacman-key --lsign-key 8A8F901A && rm sublimehq-pub.gpg
echo -e "\n[sublime-text]\nServer = https://download.sublimetext.com/arch/stable/x86_64" | sudo tee -a /etc/pacman.conf
sudo pacman -Syu sublime-text
```

##### enable swap
```
sudo systemctl enable systemd-swap
```

##### set up git and clone private repos

```
gh auth login
git config --global user.email "<email>"
git config --global user.name "<name>"
gh repo clone 12ian34/scripts
gh repo clone 12ian34/dotfiles
```

##### update xinitrc
```
cp -r ~/dotfiles/home/dotxinitrc ~/.xinitrc
```

##### start x on login
```
sudo nano /etc/profile
```
enter:
```
if [[ "$(tty)" == '/dev/tty1' ]]; then
	exec startx
fi
```
```
reboot
```

#### copy back other config

```
cp ~/dotfiles/home/10-amdgpu.conf /usr/share/X11/xorg.conf.d/
cp ~/dotfiles/home/10-monitor.conf /etc/X11/xorg.conf.d/
mkdir -p ~/.config/joplin-desktop/
cp ~/dotfiles/home/joplin-desktop/settings.json ~/.config/joplin-desktop/

cp ~/dotfiles/home/fish_history ~/.local/share/fish/
cp ~/dotfiles/home/dotgtk-bookmarks ~/.gtk-bookmarks
cp ~/dotfiles/home/dotXresources ~/.Xresources
cp -r ~/dotfiles/home/dotscreenlayout ~/.screenlayout
cp -r ~/dotfiles/home/user-dirs.dirs ~/.config/
# navigate to desired vscode settings location
cp -r ~/dotfiles/home/dotvscode .vscode

cat ~/dotfiles/home/hosts
# then append any new entries from this to /etc/hosts

cp ~/dotfiles/home/asound.conf /etc/
cp ~/dotfiles/home/vconsole.conf /etc/
cp ~/dotfiles/home/pacman.conf /etc/

cp -r ~/dotfiles/home/i3/ ~/.config/
cp -r ~/dotfiles/home/i3status-rust/ ~/.config/
cp -r ~/dotfiles/home/xfce4/ ~/.config/
cp -r ~/dotfiles/home/fish/ ~/.config/
cp -r ~/dotfiles/home/fisher/ ~/.config/
cp -r ~/dotfiles/home/nano/ ~/.config/
cp -r ~/dotfiles/home/htop/ ~/.config/
cp -r ~/dotfiles/home/picom/ ~/.config/
cp -r ~/dotfiles/home/pcmanfm/ ~/.config/
cp ~/dotfiles/home/redshift.conf ~/.config/
cp -r ~/dotfiles/home/ckb-next/ ~/.config/
cp -r ~/dotfiles/home/rofi/ ~/.config/
```

#### enable auto login
create directory:
```
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d
```
create conf:
```
sudo nano /etc/systemd/system/getty@tty1.service.d/override.conf
```
paste into `override.conf`:
```
[Service]
ExecStart=
ExecStart=-/usr/bin/agetty --autologin ian --noclear %I $TERM
```
enable and restart service
```
sudo systemctl enable getty@tty1
sudo systemctl restart getty@tty1
```

#### firefox flags (settings and hardware acceleration)
go to `about:config`

change the following preferences:

| parameter | set to value |
| --- | --- |
| browser.tabs.loadBookmarksInBackground | true |
| media.hardwaremediakeys.enabled | false |
| mousewheel.with_control.action | 1 |
| network.dns.disablePrefetch | true |
| network.http.speculative-parallel-limit | 0 |
| network.prefetch-next | false |
| gfx.webrender.all | true |
| media.ffmpeg.vaapi.enabled | true |
| media.ffvpx.enabled | false |
| media.rdd-vpx.enabled | false |
| security.sandbox.content.level | 0 |
| media.navigator.mediadatadecoder_vpx_enabled | true |
| gfx.x11-egl.force-enabled | true |
| gfx.x11-egl.force-disabled | false |

using fish, set universal permanent environment variables:
```
set -Ux MOZ_LOG PlatformDecoderModule:5
set -Ux MOZ_X11_EGL 1
set -Ux LIBVA_DRIVER_NAME radeonsi
```

#### sublime tab cycle order 'recently viewed' -> 'display order'

1. open sublime text
1. `CTRL-SHIFT-P` to open palette
1. go to `Preferences: Key Bindings`
1. Add the following to the user preferences tab (to the right of the new window):
    ```
    { "keys": ["ctrl+tab"], "command": "next_view" },
    { "keys": ["ctrl+shift+tab"], "command": "prev_view" },
    ```
#### enable and start syncthing

```
systemctl enable syncthing --user
systemctl start syncthing --user
```

#### create directories
```
mkdir -p ~/downloads/torrent/incomplete
mkdir ~/downloads/slack
mkdir ~/downloads/firefox
mkdir ~/downloads/vivaldi
mkdir ~/downloads/chromium
mkdir ~/dev
mkdir ~/pics
mkdir ~/sync
```

#### zerotier setup

```
sudo systemctl enable zerotier-one.service
sudo systemctl start zerotier-one.service
sudo zerotier-cli join <network id>
```

then login to [Zerotier](https://my.zerotier.com/) and approve the device, then check:

```
sudo zerotier-cli status
sudo zerotier-cli listnetworks
sudo zerotier-cli listpeers
```

#### insults on wrong sudo password

```
sudo nano /etc/sudoers
```
add to new line:
```
Defaults insults
```

#### VS Code extensions
- Python
- Bracket Pair Colorizer 2
- HTML snippets
- GitLens
- Prettier - Code formatter
- Material Icon Theme
- YAML
- Markdown Preview Enhanced

#### VS Code keyboard shortcuts

1. Open command palette
2. Go to `Preferences: Open Keyboard Shortcuts`
3. Paste :
    ```
    [{
        "key": "ctrl+tab",
        "command": "-workbench.action.quickOpenPreviousRecentlyUsedEditorInGroup"
    },
    {
        "key": "ctrl+tab",
        "command": "-workbench.action.quickOpenNavigateNextInEditorPicker",
        "when": "inEditorsPicker && inQuickOpen"
    },
    {
        "key": "ctrl+tab",
        "command": "workbench.action.nextEditor"
    },
    {
        "key": "ctrl+pagedown",
        "command": "-workbench.action.nextEditor"
    },
    {
        "key": "ctrl+shift+tab",
        "command": "workbench.action.previousEditor"
    },
    {
        "key": "ctrl+pageup",
        "command": "-workbench.action.previousEditor"
    },
    {
        "key": "ctrl+shift+tab",
        "command": "-workbench.action.quickOpenLeastRecentlyUsedEditorInGroup"
    },
    {
        "key": "ctrl+shift+tab",
        "command": "-workbench.action.quickOpenNavigatePreviousInEditorPicker",
        "when": "inEditorsPicker && inQuickOpen"
    }]
    ```

#### copy Thunderbird profile

mount drive containing old profile folder, navigate to its location, then copy to local
```
cp -r .thunderbird ~/
```

#### lights off
1. run `lxappearance`
1. change theme to `Adwaita-dark`
1. change default font to `Roboto Mono` `Regular` `11`
1. change icon theme to `Archdroid-Grey`

#### create python virtual environment and get some basic stuff
1. `mkdir ~/.venvs/`
1. `python -m venv ~/.venvs/venv_general`
1. `source ~/.venvs/venv_general/bin/activate.fish`
1. `pip install --upgrade pip`
1. `pip install wheel`
1. `pip install pandas numpy lxml jupyter jupyter_contrib_nbextensions jupyter_nbextensions_configurator`
1. `jupyter notebook --generate-config`
1. `jupyter notebook password`

#### Jupyter Notebook extensions and configurator
1. `source ~/.venvs/venv_general/bin/activate.fish`
1. `jupyter contrib nbextension install --user`
1. `jupyter nbextensions_configurator enable --user`
1. Configure extensions:
    1. Go to the [configurator](http://localhost:8888/tree#nbextensions_configurator)
    2. untick "disable configuration for..."
    3. Enable:
        - AutoSaveTime
        - Code prettify
        - Collapsible Headings
        - Highlight selected word
        - Notify
        - Variable Inspector
        - Codefolding
        - ExecuteTime
        - Live Markdown Preview
        - table_beautifier
        - zenmode
            - add url: https://cdn.spacetelescope.org/archives/images/screen/heic0611b.jpg
            - untick "Use builtin backgrounds...", "Hide the header..." and "Hide the menubar..."
        - Hinterland
        - Toggle all line numbers
        - Autopep8
        - Move selected cells
        - Rubberband

# sources:
- https://wiki.archlinux.org/index.php/Installation_guide
- https://linuxconfig.org/install-arch-linux-on-thinkpad-x1-carbon-gen-7-with-encrypted-filesystem-and-uefi
- https://github.com/ejmg/an-idiots-guide-to-installing-arch-on-a-lenovo-carbon-x1-gen-6
- https://gist.github.com/OdinsPlasmaRifle/e16700b83624ff44316f87d9cdbb5c94
- https://www.thelinuxsect.com/?p=36
- https://turlucode.com/arch-linux-install-guide-efi-lvm-luks/#1499978715503-ad8f936f-d6c9
- https://turlucode.com/arch-linux-install-guide-step-1-basic-installation/
- https://turlucode.com/arch-linux-install-guide-step-2-desktop-environment-installation/
- https://gist.github.com/heppu/6e58b7a174803bc4c43da99642b6094b
- https://0x00sec.org/t/arch-linux-with-lvm-on-luks-dm-crypt-disk-encryption-installation-guide-legacy-bios-system/1479
- https://gist.github.com/HardenedArray/31915e3d73a4ae45adc0efa9ba458b07
- https://gist.github.com/Thrilleratplay/93d57dbab36dc4304cd8
- https://gist.github.com/mattiaslundberg/8620837
- https://jairam.dev/technology/2018/06/21/install-arch-linux-with-encryption.html
- https://www.howtoforge.com/tutorial/how-to-install-arch-linux-with-full-disk-encryption/
- https://gist.github.com/njam/85ab2771b40ccc7ddcef878eb82a0fe9
- https://daenney.github.io/2017/11/11/arch-linux-xps-13-9360.html
- https://gist.github.com/ymatsiuk/1181b514a9c1979088bd2423a24928cf
- https://medium.com/@mudrii/arch-linux-installation-on-hw-with-i3-windows-manager-part-1-5ef9751a0be
- https://medium.com/@mudrii/arch-linux-installation-on-hw-with-i3-windows-manager-part-2-x-window-system-and-i3-installation-86735e55a0a0
