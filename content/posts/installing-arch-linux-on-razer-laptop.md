+++
title = 'Installing Arch Linux on Razer Laptop'
date = 2022-12-15T18:14:14-08:00
draft = false
summary = 'Documentation of my Arch setup on a new Razer laptop'
+++

### Create ISO

Download Etcher

Download latest Arch ISO from [Arch Linux](https://archlinux.org/download/)

### Boot from Media

Use F12 (or whatever key to enter boot menu) to boot from media.
You may need to disable secure boot.

### Connect to the internet

Connect a newtwork cable or use iwctl to connect.
```bash
iwctl
device list
station wlan0 connect wifissid
exit
```
### Partition Disk
```bash
lsblk
```
Use the path of your main drive. Mine is nvme0n1.
```bash
gdisk /dev/nvme0n1
```
Delete all existing partition with d.

Create boot partition with n.
- default number (1)
- default first sector
- last sector input `+512M`
- input `ef00` "EFI System" as the type.

Create root partition with n with default number
- default first sector
- default last sector
- input `8300` "Linux filesystem" as the type

Write Partitions
- press w to write partitions

### Setup Cryptography
```bash
cryptsetup luksFormat /dev/nvme0n1p2
```
Input `YES` and the new encryption password to encrypt the root partition.
```bash
cryptsetup open /dev/nvme0n1p2 root
```
### Format

Format the boot partition.
```bash
mkfs.fat -F32 /dev/nvme0n1p1
```
Format the main partition.
```bash
mkfs.ext4 /dev/mapper/root
```
### Mount

Mount the root parition.
```bash
mount /dev/mapper/root /mnt
```
Mount the boot partition.
```bash
mount --mkdir /dev/nvme0n1p1 /mnt/boot
```
### Swapfile

Get total number of megabytes of RAM.
The number is on the Mem row under the 'total' column.
```bash
free --mebi
```
Create the swap file. xxxx is the total amount from the previous number
multiplied by 1.5.
```bash
dd if=/dev/zero of=/mnt/swapfile bs=1M count=xxxx status=progress
```
Set permissions for swap file.
```
chmod 600 /mnt/swapfile
```
Set as the swapfile.
```bash
mkswap /mnt/swapfile
```
Turn on the swap.
```bash
swapon /mnt/swapfile
```
### Base Install

Use the following command to install Arch Linux.
```bash
pacstrap -K /mnt base base-devel linux linux-firmware neovim
```
If you have a keyring error use the following command then run the previous.
```bash
pacman -Sy
pacman -S archlinux-keyring
```
### Generate fstab
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```
### Switch Root

Switch to the newly installed Arch.
```bash
arch-chroot /mnt
```
### TZ and Lang

Setup timezone information.
```bash
ln -sf /usr/share/zoneinfo/US/Pacific /etc/localtime
```
Synchronize system clock with hardware.
```bash
hwclock --systohc
```
Setup locales.
```bash
nvim /etc/locale.gen
```
Uncomment en_US.UTF-8 UTF-8.
```bash
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```
### Hostname

Set your hostname. Replace 'arch' with the hostname you'd like.
```bash
echo 'razer' > /etc/hostname
```
Edit the hosts file.
```bash
nvim /etc/hosts
```
```bash
127.0.0.1     localhost
::1           localhost
127.0.1.1     razer.localdomain        razer
```
### Set Root Password
```bash
passwd
```
### Initial Ramdisk

Add encrypt to ramdisk.
```bash
nvim /etc/mkinitcpio.conf
```
In the HOOKS array, add `encrypt` between block and filesystems and add
`resume` between filesystems and fsck.

Generate Ramdisk.
```bash
mkinitcpio -P
```
### Boot Manager

Install EFI Boot Manager and CPU microcode. Use amd-ucode instead if AMD.
```bash
pacman -S efibootmgr intel-ucode
```
Get UUID of the device.
```bash
blkid -s UUID -o value /dev/nvme0n1p2
```
Get the offset of the swapfile. It is the first number of “physical_offset” of the line ext “0:”.
```bash
filefrag -v /swapfile | head -n 4
```
Replace “xxxx” with the UUID of the nvme0n1p2 device and “yyyy” with the 
offset of the swapfile to tell the boot manager about our encrypted file system.
```bash
efibootmgr --disk /dev/nvme0n1 --part 1 --create --label "Arch Linux" --loader
/vmlinuz-linux --unicode 'cryptdevice=UUID=xxxx:root root=/dev/mapper/root 
resume=/dev/mapper/root resume_offset=yyyy rw initrd=\intel-ucode.img 
initrd=\initramfs-linux.img' --verbose
```
#### NOTE: On an NVidia system I ran into a kernel bug that required an efistub change

Notice the ibt=off
```bash
efibootmgr --disk /dev/nvme0n1 --part 1 --create --label "Arch Linux" --loader
/vmlinuz-linux --unicode 'cryptdevice=UUID=xxxx:root root=/dev/mapper/root 
resume=/dev/mapper/root resume_offset=yyyy rw ibt=off initrd=\intel-ucode.img 
initrd=\initramfs-linux.img' --verbose
```
### Install Network Manager
```bash
pacman -S networkmanager
systemctl enable NetworkManager
```
### Reboot

Remove the install media.
```bash
exit
reboot
```
# Post-Install Setup

Use `nmcli` to connect to the internet.
```bash
nmcli device wifi list
nmcli device wifi connect WIFI password PASSWORD
```
### Verify System is up-to-date
```bash
pacman -Syu
```
### Setup new user

Uncomment `%wheel ALL=(ALL) NOPASSWD: ALL` to allow members of the wheel group 
to run privileged commands.
```bash
EDITOR=nvim visudo
```
Add a new user.
```bash
useradd --create-home --groups wheel red
```
Set user password.
```bash
passwd red
```
Exit and login as your new user.

### Setup Window Manager

Install Xorg
```bash
sudo pacman -S git libx11 libxft xorg-server xorg-xinit terminus-font libxinerama
```
Install my suckless-st
```bash
git clone https://github.com/redanthrax/st.git
cd st
make
sudo make install
```
Install my suckless-dwm
```bash
git clone https://github.com/redanthrax/dwm.git
cd dwm
make
sudo make install
```
Install suckless-dmenu
```bash
git clone git://git.suckless.org/dmenu
cd dmenu
make
sudo make install
```
Install slstatus
```bash
git clone https://git.suckless.org/slstatus
cd slstatus
make
sudo make install
```
# Additional Setup

### Yay AUR Helper
```bash
cd /opt
sudo git clone https://aur.archlinux.org/yay-git.git
sudo chown -R red:red ./yay-git
cd yay-git
makepkg -si
```
# Notifications

Install notification helper
```bash
yay -S dunst
```
### Setup shell
```bash
sudo pacman -S zsh
```
Change shell to zsh.
```bash
chsh -s /bin/zsh
```
Exit and login again to run the zsh wizard. Read and set every setting.

Install Oh-My-Zsh
```bash
yay -S oh-my-zsh-git
```
Install the PowerLevel10k theme.
```bash
yay -S zsh-theme-powerlevel10k-git
echo 'source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
```
Install FiraCode Nerd Fonts
```bash
yay -S ttf-firacode-nerd
```
Configure PowerLevel10k.
```bash
p10k configure
```
### Install display manager

Install LightDM Mini Greeter
```bash
yay -S lightdm-gtk-greeter lightdm-mini-greeter
sudo nvim /etc/lightdm/lightdm.conf
```
Add the following configurations to the config under [Seat:*]
```bash
greeter-session=lightdm-mini-greeter
user-session=dwm
session-wrapper=/etc/lightdm/Xsession
```
Modify the greeter conf and specify username.
```bash
sudo nvim /etc/lightdm/lightdm-mini-greeter.conf
```
Enable lightdm service.
```bash
sudo systemctl enable lightdm
```
Add dwm to startup application
```bash
sudo mkdir /usr/share/xsessions
sudo nvim /usr/share/xsessions/dwm.desktop
```
```bash
[Desktop Entry]
Encoding=UTF-8
Name=Dwm
Comment=The Dynamic Window Manager
Exec=dwm
Icon=dwm
Type=XSession
```

### Customize Setup

Start x server
```bash
nvim .zprofile
```
```bash
# Autostart X using startx after tty login.
if [[ -z "$DISPLAY" ]] && [[ $(tty) = /dev/tty1 ]]; then
    exec startx
fi
```
Install picom and feh.
```bash
yay -S picom feh
```
Add startup options to .xprofile.
```bash
nvim .xprofile
```
```bash
#!/bin/sh
slstatus &
picom -f -b &
feh --no-fehbg --bg-fill '/home/red/Pictures/wallpapers/wallpaper.png'
```
### Clone and link configs
```bash
git clone https://github.com/redanthrax/config.git
ln -s ~/source/config/picom ~/.config/picom 
ln -s ~/source/config/nvim ~/.config/nvim 
```
### Setup neovim
```bash
yay -S nvim-packer-git unzip
```
Open neovim and use the following command.
```bash
:PackerSync
```
Run checkhealth and resolve warnings/errors
```bash
:checkhealth
```
### Install a web browser
```bash
yay -S brave-bin
```
### Setup Sound
```bash
sudo pacman -S pipewire pipewire-pulse wireplumber
```
### Setup Bluetooth
```bash
sudo pacman -S bluez bluez-utils
sudo systemctl enable bluetooth.service --now
```
### Lock Root User
```bash
sudo passwd --lock root
```
### Install Firewall
```bash
sudo pacman -S nftables
sudo systemctl enable nftables.service --now
```
### Enable Time Synchronization
```bash
sudo systemctl enable systemd-timesyncd.service --now
```
### Improve Power Management
```bash
sudo pacman -S tlp tlp-rdw
sudo systemctl enable tlp.service --now
sudo systemctl enable NetworkManager-dispatcher.service --now
sudo tlp-stat
```
### Enable Scheduled fstrim
```bash
sudo systemctl enable fstrim.timer --now
```
### Enable Scheduled Mirrorlist
```bash
sudo pacman -S reflector
sudo systemctl enable reflector.timer --now
```
### Reduce Swappiness
```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
```
### Setup Videocard

Install drivers
```bash
sudo pacman -S nvidia nvidia-utils nvidia-settings \
xorg-server-devel opencl-nvidia nvidia-prime
```
Verify nouveau are blacklisted.
```bash
cat /usr/lib/modprobe.d/nvidia.conf
```
Add nvidia to initramfs.
```bash
sudo nvim /etc/mkinitcpio.conf
```
```bash
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```
Rebuild initramfs
```bash
mkinitcpio -P
```
Setup pacman hook to rebuild initramfs.
```bash
sudo mkdir /etc/pacman.d/hooks
sudo nvim /etc/pacman.d/hooks/nvidia.hook
```
```bash
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia
Target=linux

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```
Install optimus-manager and optimus-manager-qt
```bash
yay -S optimus-manager optimus-manager-qt
```
Get the BusID of the nvidia card.
```bash
lspci | grep -E "VGA|3D"
```
Create optimus file.
```bash
sudo nvim /etc/X11/xorg.conf.d/optimus.conf
```
```bash
Section "Module"
    Load "modesetting"
EndSection
Section "Device"
    Identifier "nvidia"
    Driver "nvidia"
    BusID "PCI:1:0:0"
    Option "AllowEmptyInitialConfiguration"
EndSection
```
Nvidia DRM
```bash
sudo nvim /etc/modprobe.d/nvidia-drm.conf
```
```bash
options nvidia_drm modeset=1
```
Add LightDM Script.
```bash
sudo nvim /usr/local/bin/optimus.sh
```
```bash
#!/bin/sh
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --auto
```
Make the script executable.
```bash
sudo chmod a+rx /usr/local/bin/optimus.sh
```
Edit the LightDM config.
```bash
nvim /etc/lightdm/lightdm.conf
```
```bash
display-setup-script=/usr/local/bin/optimus.sh
```
Regenerate initramfs
```bash
sudo mkinitcpio -P
```
Reboot

### Network Manager Applet
```bash
yay -S network-manager-applet
```
### Update .xprofile
```bash
nvim .xprofile
```
```bash
#!/bin/sh
prime-offload && optimus-manager-qt &
/usr/local/bin/slstatus &
/usr/bin/picom -f -b &
/usr/bin/nm-applet &
```
### Set Dark Mode

Install the package xfce4-settings
```bash
sudo pacman -S xfce4-settings
```
Set your default theme to Adwaita-dark
```bash
xfconf-query -c xsettings -p /Net/ThemeName -s "Adwaita-dark"
gsettings set org.gnome.desktop.interface color-scheme prefer-dark
```
### Multi Monitor Setup

I have my 1080p gaming monitor setup next to my 1440p laptop.
DPI is still an issue in linux so I've set my DPI via .Xresources file.
```bash
Xft.dpi: 124
```
Here is my layout.sh file that I run on login to get my monitor layout to look
correct. The idea is to transform the smaller screen to a larger resolution
to match the laptop monitor. Then use a DPI that I find readable.
```bash
xrandr --output eDP-1-1 --mode 2560x1440 --pos 2496x0 --rotate normal
xrandr --output HDMI-0 --mode 1920x1080 --transform 1.3,0,0,0,1.3,0,0,0,1 --fb 2496x1404
```
You may need to run it twice.
