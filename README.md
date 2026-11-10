
# PHASE 1 PARTITIONING

# 1. Increase terminal font size for better readability on screen (Choose one)
$ setfont ter-132b
$ setfont ter-v22b

# 2. Check internet connection (LAN ONLY)
$ ping -c 4 ping.archlinux.org

> IF YOU USE WIFI please read this link
( https://wiki.archlinux.org/title/Iwd#iwctl )


# 3. Check time synchronization
$ timedatectl status
$ timedatectl set-ntp true

# 4. Check available storage devices (lsblk or fdisk)
$ lsblk -f 
$ fdisk -l

# 5. Partition NVMe disk (/dev/nvme0n1) using GPT schema
$ sgdisk --zap-all /dev/nvme0n1
$ parted -s /dev/nvme0n1 mklabel gpt

# Partition 1: EFI System Partition (1 GiB)
$ parted -s /dev/nvme0n1 mkpart ESP fat32 1MiB 1024MiB
$ parted -s /dev/nvme0n1 set 1 esp on

# Partition 2: Swap Partition (4 GiB)
$ parted -s /dev/nvme0n1 mkpart primary linux-swap 1024MiB 5120MiB

# Partition 3: Root Btrfs Partition (Remaining Space)
$ parted -s /dev/nvme0n1 mkpart primary btrfs 5120MiB 100%

# Verify partition layout
$ parted /dev/nvme0n1 print
$ lsblk -f

# 6. Format partitions
$ mkfs.fat -F32 -n "EFI" /dev/nvme0n1p1
$ mkswap -L "ARCH_SWAP" /dev/nvme0n1p2
$ swapon /dev/nvme0n1p2
$ mkfs.btrfs -f -L "ARCH_LINUX" /dev/nvme0n1p3

# 7. Create Btrfs subvolumes (Modern standard)
$ mount /dev/nvme0n1p3 /mnt
$ btrfs subvolume create /mnt/@
$ btrfs subvolume create /mnt/@home
$ btrfs subvolume create /mnt/@snapshots
$ btrfs subvolume create /mnt/@var_log
$ btrfs subvolume create /mnt/@pkg
$ btrfs subvolume list /mnt
$ umount /mnt

# 8. Mount subvolumes with SSD performance options & zstd compression
$ BTRFS_OPTS="noatime,compress=zstd:1,discard=async"

# Mount the root subvolume first
$ mount -o "$BTRFS_OPTS,subvol=@" /dev/nvme0n1p3 /mnt

# Create mount point directory structure
$ mkdir -p /mnt/boot/efi
$ mkdir -p /mnt/home
$ mkdir -p /mnt/.snapshots
$ mkdir -p /mnt/var/log
$ mkdir -p /mnt/var/cache/pacman/pkg
$ mkdir -p /mnt/mnt/wdblue

# Mount the remaining Btrfs subvolumes
$ mount -o "$BTRFS_OPTS,subvol=@home" /dev/nvme0n1p3 /mnt/home
$ mount -o "$BTRFS_OPTS,subvol=@snapshots" /dev/nvme0n1p3 /mnt/.snapshots
$ mount -o "$BTRFS_OPTS,subvol=@var_log" /dev/nvme0n1p3 /mnt/var/log
$ mount -o "$BTRFS_OPTS,subvol=@pkg" /dev/nvme0n1p3 /mnt/var/cache/pacman/pkg

# Mount EFI partition & secondary storage (WD Blue HDD)
$ mount /dev/nvme0n1p1 /mnt/boot/efi
$ mount /dev/sda1 /mnt/mnt/wdblue

# 9. Optimize mirrorlist (Indonesia & Singapore) using Reflector
$ reflector \
  --country Indonesia,Singapore \
  --protocol https \
  --latest 10 \
  --sort rate \
  --save /etc/pacman.d/mirrorlist

# 10. Install base packages & multi-option kernels (Linux, LTS, Zen) + Microcode
$ pacstrap -K /mnt \
  base \
  base-devel \
  linux \
  linux-headers \
  linux-lts \
  linux-lts-headers \
  linux-zen \
  linux-zen-headers \
  linux-firmware \
  amd-ucode \
  btrfs-progs \
  neovim \
  git \
  nano \
  networkmanager \
  sudo \
  reflector

# 11. Generate UUID-based fstab
$ genfstab -U /mnt >> /mnt/etc/fstab

# Inspect generated fstab (Optional via nano)
$ nano /mnt/etc/fstab


# PHASE 2 ARCH-CHROOT
# Enter the new chroot environment
$ arch-chroot /mnt

# 1. Configure timezone & system clock
$ ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
$ hwclock --systohc

# 2. Configure system language / locale
$ nano /etc/locale.gen
# -> (Find and uncomment the line: en_US.UTF-8 UTF-8)

$ locale-gen
$ echo "LANG=en_US.UTF-8" > /etc/locale.conf

# 3. Configure hostname & hosts file (Using 127.0.1.1 for hostname as per Arch Wiki)
$ echo "archlinux" > /etc/hostname

$ nano /etc/hosts
# -> (Manually fill the hosts file:
# 127.0.0.1   localhost
# ::1         localhost
# 127.0.1.1   archlinux.localdomain archlinux
# )

# 4. Create user, password, & Sudo privileges
$ useradd -m -G wheel -s /bin/bash famouzak
$ passwd famouzak
$ passwd  # (Set root password)

# Open visudo to grant privileges to the wheel group
$ EDITOR=nano visudo
# -> (Find the line: # %wheel ALL=(ALL:ALL) ALL -> Uncomment it by removing the #)

# 5. Configure pacman.conf (Multilib, Color, ParallelDownloads, & ILoveCandy)
$ nano /etc/pacman.conf
# -> (Perform manual editing:
# - Change #ParallelDownloads = 5 to ParallelDownloads = 10
# - Enable Color and add the ILoveCandy line below it
# - Uncomment the [multilib] block and the Include line below it)

# Resynchronize repository database
$ pacman -Syu


# PHASE 3 INSTALL PACKAGES

# 1. Install KDE Plasma desktop environment & supporting utilities
$ pacman -S --needed \
  git networkmanager pacman-contrib \
  alacritty plasma-meta \
  gwenview okular konsole dolphin kdeconnect kdenlive \
  sddm wayland egl-wayland xdg-user-dirs \
  qt6-base qt6-declarative qt6-svg

# 2. Install GRUB & backup tools (Tanpa snap-pac dulu)
$ pacman -S --needed \
  grub efibootmgr grub-btrfs snapper \
  inotify-tools os-prober


# PHASE 4 SYSTEM CONFIGURATION

# === A. SDDM & ELEGANT THEME CONFIGURATION ===
$ git clone https://github.com/sniper1720/elegant-sddm-archlinux-theme.git /tmp/elegant-sddm
$ mkdir -p /usr/share/sddm/themes/
$ cp -r /tmp/elegant-sddm/elegant-archlinux /usr/share/sddm/themes/
$ rm -rf /tmp/elegant-sddm

$ mkdir -p /etc/sddm.conf.d
$ nano /etc/sddm.conf.d/theme.conf
# -> (Populate theme.conf file:
# [Theme]
# Current=elegant-archlinux
# )


# === B. GRUB BOOTLOADER & HYPERFLUENT THEME CONFIGURATION ===
$ mkdir -p /boot/grub/themes
$ git clone --depth=1 \
  https://github.com/Coopydood/HyperFluent-GRUB-Theme.git \
  /tmp/hyperfluent-grub
$ cp -r /tmp/hyperfluent-grub/arch \
  /boot/grub/themes/HyperFluent
$ rm -rf /tmp/hyperfluent-grub

$ nano /etc/default/grub
# -> (Manually adjust GRUB parameters:
# GRUB_TIMEOUT=3
# GRUB_GFXMODE=1920x1080x32,auto
# GRUB_DISABLE_SUBMENU=y
# GRUB_TERMINAL_OUTPUT="gfxterm"
# GRUB_THEME="/boot/grub/themes/HyperFluent/theme.txt"
# GRUB_TOP_LEVEL="/boot/vmlinuz-linux-zen"
# )


# === C. MKINITCPIO CONFIGURATION & GRUB INSTALLATION ===
$ nano /etc/mkinitcpio.conf
# -> (Find the MODULES=() line and change it to:
# MODULES=(btrfs amdgpu)
# )

$ mkinitcpio -P

# Install GRUB to EFI (with --removable option for extra safety)
$ grub-install \
  --target=x86_64-efi \
  --efi-directory=/boot/efi \
  --bootloader-id=GRUB \
  --removable

$ grub-mkconfig -o /boot/grub/grub.cfg


# === D. ACTIVATE SYSTEMD SERVICES ===
$ systemctl enable NetworkManager
$ systemctl enable sddm
$ systemctl enable grub-btrfsd

# Exit chroot environment, unmount partitions, and reboot system
$ exit
$ umount -R /mnt
$ reboot

# POST INSTALL SETUP

# === A. SETUP SNAPPER & SNAP-PAC (Aman setelah sistem aktif) ===
# 1. Temporarily unmount and clear default snapper folder if needed
$ sudo umount /.snapshots || true
$ sudo rm -rf /.snapshots

# 2. Create snapper configuration for root btrfs
$ sudo snapper -c root create-config /

# 3. Re-create proper subvolume mount configuration
$ sudo btrfs subvolume delete /.snapshots
$ sudo mkdir -p /.snapshots
$ sudo mount -a

# 4. Set permission & allow user access
$ sudo chmod 750 /.snapshots
$ sudo chown :wheel /.snapshots
$ sudo snapper -c root set-config ALLOW_USERS=famouzak SYNC_USER=yes

# 5. Enable snapper automation timers
$ sudo systemctl enable --now snapper-timeline.timer
$ sudo systemctl enable --now snapper-cleanup.timer

# 6. Install snap-pac now that snapper configuration is fully ready
$ sudo pacman -S --needed snap-pac


# === B. INSTALL YAY (AUR HELPER) ===
$ git clone https://aur.archlinux.org/yay.git /tmp/yay
$ cd /tmp/yay
$ makepkg -si --noconfirm
$ cd ~
$ rm -rf /tmp/yay


# === C. INSTALL GAMING, MULTIMEDIA & AMD DRIVERS ===
$ sudo pacman -S --needed \
  mesa lib32-mesa \
  vulkan-radeon lib32-vulkan-radeon \
  libva-mesa-driver lib32-libva-mesa-driver \
  lib32-vulkan-icd-loader vulkan-icd-loader vulkan-tools \
  pipewire pipewire-pulse pipewire-alsa pipewire-jack \
  wireplumber pavucontrol easyeffects \
  lsp-plugins-lv2 calf alsa-utils \
  steam wine-staging winetricks \
  gamemode lib32-gamemode gamescope \
  mangohud lib32-mangohud goverlay lact \
  obs-studio obs-studio-plugin-browser \
  ffmpeg vlc mpv gstreamer \
  gnutls lib32-gnutls giflib \
  v4l2loopback-dkms v4l2loopback-utils \
  flatpak

# Configure Gamemode & verification
$ sudo usermod -aG gamemode famouzak
# (Logout then log back in, then test with command:)
$ gamemoded -t

# Enable LACT service (AMD GPU Controller)
$ sudo systemctl enable --now lactd

# Check DKMS module status
$ dkms status


# === D. INSTALL AUR PACKAGES ===
$ yay -S --needed \
  brave-bin \
  darkly-bin \
  protonplus \
  lib32-giflib


# === E. FLATPAK CONFIGURATION (System-wide consistent with sudo) ===
$ sudo flatpak remote-add --if-not-exists \
  flathub https://dl.flathub.org/repo/flathub.flatpakrepo

$ sudo flatpak install flathub org.vinegarhq.Sober

