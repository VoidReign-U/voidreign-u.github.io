---
title: arch_installation
author: VoidReign-U
date: 2025-10-28 00:00:00 +0800
categories:
  - Linux
tags:
  - System
permalink: /System/ArchInstall
---

# Arch Linux Installation (EFI Only)

---

## 1. Boot the ISO

Flash the Arch ISO to USB and boot in UEFI mode.

Check internet:

```bash
ping archlinux.org
```

Wi‑Fi users:

```bash
iwctl
station wlan0 connect "SSID"
# or
station wlo1 connect "SSID"
```

---

## 2. Select Disk

Verify your disk:

```bash
lsblk
```

---

## 3. Partition (GPT)

You can use `cfdisk` for manual partitioning or this automated method (replace `sda` with your drive):

```bash
# WARNING: This erases all data on /dev/sda
sgdisk -Z /dev/sda
sgdisk -n1:0:+512M -t1:ef00 -c1:"EFI System" /dev/sda
sgdisk -n2:0:0      -t2:8300 -c2:"Linux Root" /dev/sda
```

| Partition | Size         | Type             |
|-----------|--------------|------------------|
| EFI       | 512M - 1G    | FAT32            |
| ROOT      | Rest of disk | Linux filesystem |

---

## 4. Format Partitions

### 4.0 Before you format — optional: encrypt the root (recommended for daily use)

Only the EFI partition stays unencrypted so GRUB can start.

### 4.1 Encrypt Root with LUKS (optional but recommended)

```bash
# Initialize LUKS on the root partition
cryptsetup luksFormat /dev/sda2

# Open it and map to /dev/mapper/cryptroot
cryptsetup open /dev/sda2 cryptroot
```

> Use a strong passphrase. Keep a backup of the LUKS header with `cryptsetup luksHeaderBackup` if the data matters.

### 4.2 Format partitions

Use `/dev/mapper/cryptroot` instead of `/dev/sda2` if encrypted.

- **EFI partition**:

```bash
mkfs.fat -F32 /dev/sda1
```

- **Root partition** (choose one):

```bash
# ext4
mkfs.ext4 /dev/mapper/cryptroot

# or btrfs
mkfs.btrfs /dev/mapper/cryptroot
```

> Recommended: **btrfs** for snapshots, compression, and advanced features.

---

## 5. Mount File Systems

### Ext4

```bash
mount /dev/mapper/cryptroot /mnt   # or /dev/sda2 if not encrypted
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
```

### Btrfs (with subvolumes)

```bash
# Temporarily mount to create subvolumes
mount /dev/mapper/cryptroot /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
umount /mnt

# Mount with recommended options (SSD example)
mount -o noatime,compress=zstd:3,ssd,space_cache=v2,subvol=@ /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{home,boot/efi}
mount -o noatime,compress=zstd:3,ssd,space_cache=v2,subvol=@home /dev/mapper/cryptroot /mnt/home
mount /dev/sda1 /mnt/boot/efi
```

> Remove the `ssd` option for HDDs. Adjust compression level as desired.

- like This
### For SSD

```sh
# mount with subvol options (SSD example)
mount -o noatime,compress=zstd:3,ssd,space_cache=v2,subvol=@ /dev/sda2 /mnt
mkdir -p /mnt/{home,boot/efi}
mount -o noatime,compress=zstd:3,ssd,space_cache=v2,subvol=@home /dev/sda2 /mnt/home
mount /dev/sda1 /mnt/boot/efi

```

### For non-SSD
```sh
mount -o rw,noatime,compress=zstd:3,autodefrag,space_cache=v2,subvol=@ /dev/sda2 /mnt
mkdir -p /mnt/{home,boot/efi}
mount -o rw,noatime,compress=zstd:3,autodefrag,space_cache=v2,subvol=@home /dev/sda2 /mnt/home
mount /dev/sda1 /mnt/boot/efi
```
---

## 6. Base Installation

Update mirrorlist for speed (optional):

```bash
reflector --save /etc/pacman.d/mirrorlist --latest 7 --protocol https
```

Choose kernel and install base packages (add `btrfs-progs` if you used btrfs):

```bash
# Example: linux-lts
pacstrap -K /mnt base linux-lts linux-lts-headers linux-firmware neovim sudo git reflector btrfs-progs

# or latest linux
pacstrap -K /mnt base linux linux-headers linux-firmware neovim sudo git reflector
```

---

## 7. Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Review `/mnt/etc/fstab` briefly to ensure correct mount points.

---

## 8. Chroot

```bash
arch-chroot /mnt
```

---

## 9. Time & Locale

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc

# locale
nvim /etc/locale.gen   # Uncomment en_US.UTF-8 UTF-8 (or your locale)
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

---

## 10. Bootloader & System Packages (GRUB)

Install required packages:

```bash
pacman -Syu
pacman -S base-devel grub efibootmgr mtools networkmanager reflector openssh iptables-nft ipset
# add grub-btrfs only if using btrfs snapshots
pacman -S --needed grub-btrfs   # optional
```

### Configure mkinitcpio for LUKS / Btrfs

Edit `/etc/mkinitcpio.conf`:

- Add `encrypt` to `HOOKS` **before** `filesystems`:

```
HOOKS=(base udev autodetect modconf block encrypt filesystems keyboard fsck)
```

- Add modules for early decryption and btrfs if needed:

```
MODULES=(usbhid atkbd btrfs)
```

Regenerate initramfs:

```bash
mkinitcpio -P
```

### Install GRUB (UEFI)

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
# Optional if using custom Secure Boot keys:
# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --disable-shim-lock
```

Get UUID of the encrypted partition (if used):

```bash
blkid /dev/sda2
```

Edit `/etc/default/grub` and add crypt device in the kernel cmdline:

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=<UUID-of-sda2>:cryptroot root=/dev/mapper/cryptroot"
```

Then generate GRUB config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

(Optional) Add `/etc/crypttab` for clarity:

```bash
echo "cryptroot UUID=$(blkid -s UUID -o value /dev/sda2) none luks" > /etc/crypttab
```

---

## 11. User & Sudo

```bash
useradd -m -G wheel -s /bin/bash USERNAME
passwd USERNAME
mkdir -p /etc/sudoers.d
chmod 755 /etc/sudoers.d
EDITOR=nvim visudo -f /etc/sudoers.d/USERNAME
# then add the line: USERNAME ALL=(ALL) ALL
chmod 0440 /etc/sudoers.d/USERNAME
```

> Using `visudo -f` verifies the file syntax before saving.

---

## 12. Apps Setup (choose what fits you)

### i3 + common apps

```bash
sudo pacman -S i3 kitty xorg xdg-user-dirs rofi firefox feh acpid bluez bluez-utils thunar obsidian cherrytree flameshot mpv ttf-jetbrains-mono ttf-firacode-nerd noto-fonts
```

### Hyprland (Wayland)

```bash
sudo pacman -S hyprland kitty xorg-xwayland waybar xdg-user-dirs rofi firefox feh acpid bluez bluez-utils thunar obsidian hyprshot hyprlock hyprpolkitagent swww mpv ttf-jetbrains-mono ttf-firacode-nerd noto-fonts
```

### XFCE / Gnome (examples)

```bash
sudo pacman -S xfce4 xfce4-goodies
sudo pacman -S gnome gnome-extra
```

### PolicyKit for GUI Privileges (for tiling window managers only)
- PolicyKit (polkit) enables GUI applications to safely request administrative (root) privileges, ensuring secure and user-friendly system management.

- For tiling window managers, installing a PolicyKit agent ensures that GUI applications can request administrative (root) privileges even without a full desktop environment.

### Install PolicyKit Agent for LXQt:

```sh
sudo pacman -S lxqt-policykit
```

### Install PolicyKit Agent for KDE Plasma:

```sh
sudo pacman -S polkit-kde-agent
```

### Install PolicyKit Agent for XFCE:

```sh
sudo pacman -S xfce4-polkit-plugin
```

### Install PolicyKit Agent for GNOME:

```sh
sudo pacman -S gnome-polkit
```

### Install PolicyKit Agent for MATE:

```sh
sudo pacman -S mate-polkit
```

### Install PolicyKit Agent for Cinnamon:

```sh
sudo pacman -S cinnamon-polkit
```

### Install PolicyKit Agent for Budgie:

```sh
sudo pacman -S budgie-polkit
```

> Note: This will allow password prompts for actions like mounting drives, network changes, and system tools.
> Note: But in my config for i3 i use lxqtpolkit u need to edit it if you choose some thing else for example version by gnome xfce ect ect

### Audio: PipeWire (recommended) or PulseAudio (legacy)

```bash
# PipeWire
sudo pacman -S pipewire pipewire-alsa pipewire-jack pipewire-pulse wireplumber

# PulseAudio (if needed)
sudo pacman -S pulseaudio pulseaudio-alsa pulseaudio-utils
```

If laptop: `pacman -S upower acpi acpid`

---

## 13. Enable Services

```bash
systemctl enable NetworkManager
systemctl enable sshd
systemctl enable bluetooth
systemctl enable reflector.timer
systemctl enable fstrim.timer
systemctl enable acpid
```

For PipeWire (user):

```bash
systemctl --user enable --now pipewire
```

---

## 14. GPU & CPU Firmware/Drivers

```bash
# Intel microcode
pacman -S intel-ucode

# Intel GPU (if using Xorg fallback)
pacman -S xf86-video-intel

# AMD microcode
pacman -S amd-ucode

# AMD GPU
pacman -S xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon

# NVIDIA (proprietary recommended for modern cards)
pacman -S nvidia nvidia-utils lib32-nvidia-utils
# or for DKMS
pacman -S nvidia-dkms nvidia-utils
```

> Avoid `xf86-video-nouveau` for modern NVIDIA cards — prefer `nvidia` or `nvidia-dkms`.

---

## 15. Display/Login Managers

Examples:

```bash
# ly (simple)
pacman -S ly
systemctl enable ly.service

# sddm
pacman -S sddm
systemctl enable sddm.service

# lightdm
pacman -S lightdm
systemctl enable lightdm.service

# gdm
pacman -S gdm
systemctl enable gdm.service
```

---

## 16. Timeshift (Snapshots) + Autosnapshots (optional)

```bash
# AUR helper required
paru -S timeshift timeshift-autosnap
sudo timeshift --list-devices
sudo timeshift --create --comments "$(date +%F) Initial snapshot" --tags D
```

If using `grub-btrfs` edit service as needed and regenerate grub config.

---

## 17. Power Improvements

```bash
sudo pacman -S power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon.service

# Or auto-cpufreq from AUR
paru -S auto-cpufreq
sudo systemctl enable --now auto-cpufreq.service
```

---

## 18. Optional: BlackArch repo & pentest tools

*Use with caution — mixing repos can affect stability.*

```bash
curl -O https://blackarch.org/strap.sh
# verify checksum as instructed on blackarch site
chmod +x strap.sh
sudo ./strap.sh
sudo pacman -Syu
```

Example security tools:

```bash
sudo pacman -S seclists metasploit feroxbuster gobuster curl wget burpsuite sqlmap wireshark-cli binwalk radare2 gdb-multiarch
```

---

### Final Steps — Exit & Reboot

```bash
exit
umount -R /mnt
reboot
```

---

### After First Boot

- Clone dotfiles (optional):

```bash
git clone https://github.com/VoidReign-U/dotfiles ~/.dotfiles
```

- Ensure user dirs exist:

```bash
xdg-user-dirs-update
mkdir -p ~/Pictures/Screenshots ~/Pictures/Wallpapers
```

- Neovim setup (optional):

```bash
git clone https://github.com/VoidReign-U/nvim.git "${XDG_CONFIG_HOME:-$HOME/.config}"/nvim
sudo pacman -S neovim go python-pip ripgrep fd curl wget git unzip nodejs npm
```

---
## 19. Security

The following steps form the foundation of a secure Arch system. You can take additional steps to harden your system further, but consider this a starting point.

---

### Secure Boot

#### Step 1 — Prepare BIOS/UEFI

1. Boot into your system BIOS/UEFI.
    
2. Clear vendor keys (if not dual-booting).
    
3. Disable Secure Boot temporarily.
    
4. Ensure that **Setup Mode** is enabled.
    

---

#### Step 2 — Configure Secure Boot in Arch

1. Install **sbctl**:
    

```bash
sudo pacman -Syu sbctl
```

2. Generate keys:
    

```bash
sudo sbctl create-keys
```

3. Enroll your keys:
    

```bash
sudo sbctl enroll-keys
```

> **Note:** If you are using other kernel versions (e.g., `linux-zen` or `linux-lts`), you will need to sign those kernels too.

4. Reinstall GRUB with TPM module and custom keys:
    

```bash
sudo grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --disable-shim-lock --modules="tpm" --recheck
```

> `--disable-shim-lock` is optional and should only be used when using custom keys (not Microsoft-signed shim).

5. Generate GRUB configuration:
    

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

6. Sign necessary binaries:
    

```bash
sudo sbctl sign -s /boot/vmlinuz-linux
sudo sbctl sign -s /boot/EFI/GRUB/grubx64.efi
```

---

#### Step 3 — Harden GRUB

1. Set a GRUB password:
    

```bash
grub-mkpasswd-pbkdf2
```

- Example:
    

```bash
set superusers="admin"
password_pbkdf2 admin <generated-hash>
```

2. Restrict access to GRUB files:
    

```bash
sudo chmod 600 /boot/grub/grub.cfg
sudo chmod -R 700 /etc/grub.d
```

3. Prevent modifications to GRUB config:
    

```bash
sudo chattr +i /boot/grub/grub.cfg
```

4. Update `/etc/default/grub` for security:
    

```bash
GRUB_DISABLE_RECOVERY="true"
GRUB_DISABLE_OS_PROBER="true"
```

5. Regenerate GRUB configuration after changes:
    

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

#### Step 4 — Reboot & Test

1. Reboot the machine.
    
2. Re-enable Secure Boot in BIOS.
    
3. Ensure the system boots correctly and GRUB password works.
    

---
your Arch system now has **Secure Boot** configured with a custom GRUB password and hardened configuration.
# Notes & Tips

- Always backup important data before partitioning.
- Keep a **backup of the LUKS header** with `cryptsetup luksHeaderBackup`.
- If Secure Boot is enabled, you'll need to handle module signing or use `shim`/`mok`.
- Prefer `visudo -f /etc/sudoers.d/USERNAME` when adding sudo rules to avoid syntax errors.

---

Done — enjoy your clean and powerful Arch Linux setup.
