---
title: How To Install My Version Of Arch
author: VoidReign-U
date: 2025-10-28 00:00:00 +0800
categories:
  - Linux
tags:
  - System
permalink: /System/ArchInstall
---
# Arch Linux Installation (My Way Only EFI)

This is my full personal guide for installing Arch Linux with Btrfs, PipeWire, Bluetooth, i3, my configs, and hacking tools support. Simple and clean.

---

## 1. Boot the ISO
Flash Arch ISO to USB, boot in UEFI mode.

Check internet:

```sh
ping archlinux.org
```

WiFi users:

```sh
iwctl
station wlan0 connect "SSID"

or 

station wlo1 connect "SSID"
```

---

## 2. Select Disk
Make sure you select the correct drive:

```sh
lsblk
```

---

## 3. Partition (GPT)

| Partition | Size | Type |
|----------|------|------|
| EFI      | 512M - 1G | FAT32 |
| ROOT     | Rest of disk | Linux filesystem |

---

## 4. Format
```sh
mkfs.fat -F32 /dev/sda1
mkfs.btrfs /dev/sda2
```

---

## 5. Btrfs Subvolumes

```sh
mount /dev/sda2 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
umount /mnt
```

---

## 6. Mount File Systems

### For SSD

`mount -o rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@ /dev/sda2 /mnt mkdir -p /mnt/{home,boot/efi} mount -o rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@home /dev/sda2 /mnt/home mount /dev/sda1 /mnt/boot/efi`

### For non-SSD

`mount -o rw,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=@ /dev/sda2 /mnt mkdir -p /mnt/{home,boot/efi} mount -o rw,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=@home /dev/sda2 /mnt/home mount /dev/sda1 /mnt/boot/efi`

> **Note:** The `ssd` option is optional. Only use it if your device is actually an SSD; otherwise, omit it to avoid warnings or errors.```
---

## 7. Base Installation

```sh
pacstrap -K /mnt base linux-lts linux-lts-headers linux-firmware neovim sudo btrfs-progs git reflector
```

---

## 8. Generate fstab
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

---

## 9. Chroot
```sh
arch-chroot /mnt
```

---

## 10. Time & Locale

```sh
ln -sf /usr/share/zoneinfo/Continent/City /etc/localtime
hwclock --systohc
```

Enable English UTF-8:

```sh
nvim /etc/locale.gen
# Uncomment: en_US.UTF-8 UTF-8
locale-gen
```

---

## 11. Bootloader With NetwokManager (GRUB)

```sh
pacman -S base-devel grub efibootmgr grub-btrfs mtools networkmanager reflector
openssh iptables-nft ipset ```

Install GRUB:

```sh
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 12. User + Sudo

```sh
useradd -m -G wheel yourname
passwd yourname
mkdir /etc/sudoers.d
chmod 755 /etc/sudoers.d
echo "yourname ALL=(ALL) ALL" >> /etc/sudoers.d/yourname
chmod 0440 /etc/sudoers.d/yourname
```

---

## 13. Enable Services

```sh
systemctl enable NetworkManager
systemctl enable sshd
systemctl enable bluetooth
systemctl enable reflector.timer
systemctl enable fstrim.timer
systemctl enable acpid
```

PipeWire:

```sh
systemctl --user enable --now pipewire
```

---

## 14. GPU Drivers (pick your hardware)

```sh
# Intel
pacman -S xf86-video-intel

# AMD
pacman -S xf86-video-amdgpu

# NVIDIA
pacman -S xf86-video-nouveau
```

---

## 15. Enable Login Manager (LY)

```sh
pacman -S ly
systemctl enable ly.service
```

---

## 16. Exit & Reboot

```sh
exit
umount -R /mnt
reboot
```

---

# After First Boot

### Log by using shell from options 

## i3 + Apps Setup

```sh
sudo pacman -S i3 kitty xorg rofi firefox feh upower git acpid bluez bluez-utils pipewire pipewire-alsa pipewire-pulse wireplumber thunar obsidian cherrytree flameshot yazi mpv ttf-jetbrains-mono ttf-firacode-nerd ttf-noto-nerd
```

---

## My Dotfiles (i3 + Zsh + Nvim)

```sh
git clone https://github.com/VoidReign-U/dotfiles ~/.dotfiles
```

Copy configs you like.

Make sure user directories are created:

```sh
xdg-user-dirs-update
```

Create Wallpapers and Screenshots folders for your workflow:

```sh
mkdir -p ~/Pictures/Screenshots
mkdir -p ~/Pictures/Wallpapers
```

You can now set wallpapers and screenshots paths properly in your i3 config.

---
### NvChad Setup (Optional)

My **NvChad config** is already prepared with everything mentioned above:

- Preconfigured **theme**  
- **Autocomplete / IntelliSense** for Bash, Python, HTML, CSS, PHP, Perl, Go, Rust, etc.  
- All custom **keymaps, plugins, and LSP settings** as I use them  

To use it:

```sh
# Copy my ready config
git clone https://github.com/VoidReign-U/dotfiles ~/.dotfiles
cp -r ~/.dotfiles/.config/nvim/* ~/.config/nvim/
nvim
```

> Note: This is optional. You can skip it if you want your own NvChad setup, but everything is ready for you in my config.

---
## PolicyKit for GUI Privileges (Required for i3)

Install LXQt PolicyKit Agent:

```sh
sudo pacman -S lxqt-policykit
```

This will allow password prompts for actions like mounting drives, network changes, and system tools.

---

## Btrfs Snapshots (Highly Recommended)

```sh
yay -S timeshift timeshift-autosnap
sudo timeshift --list-devices
sudo timeshift --create --comments "[Date] Initial snapshot" --tags D
sudo systemctl edit --full grub-btrfsd
```

Inside editor:

Replace:
```
ExecStart=/usr/bin/grub-btrfsd --syslog /.snapshots
```
With:
```
ExecStart=/usr/bin/grub-btrfsd --syslog -t
```

Then:
```sh
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## Power Improvements

```sh
sudo pacman -S power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon.service
yay -S auto-cpufreq
sudo systemctl enable --now auto-cpufreq.service
```

---

## Ready To Hack

BlackArch repo:

```sh
curl -O https://blackarch.org/strap.sh
echo e26445d34490cc06bd14b51f9924debf569e0ecb strap.sh | sha1sum -c
chmod +x strap.sh
sudo ./strap.sh
sudo pacman -Syu
```

Containers (Podman + Distrobox):

```sh
sudo pacman -S podman podman-docker podman-compose distrobox
```

QEMU/KVM:

```sh
sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils \
openbsd-netcat ebtables nftables libguestfs
sudo usermod -a -G libvirt $(whoami)
sudo systemctl enable --now libvirtd
```
## Make Libvirt Group for your user

Edit `/etc/libvirt/libvirtd.conf` (Change the following Lines)

```fallback
unix_sock_group = "libvirt"
unix_sock_rw_perms = "0770"
```

VirtualBox:

```sh
sudo pacman -S virtualbox virtualbox-guest-iso virtualbox-host-dkms
```

---

## Distrobox (Parrot / Kali) — Use full distros like they are your main system

If you want isolated environments for pentesting or compatibility, create Distroboxes using Podman. I use two flavours: a **root** Distrobox (full root inside, useful for apt/pacman ops) and a **non-root** Distrobox (normal user inside, no sudo prompt inside container). Keep names unique.

> NOTE: Make sure `podman` and `distrobox` are installed (see earlier). These commands assume Podman rootless is available.

### Create root Distroboxes (root inside container — useful when you need full control)
```sh
# Parrot as root container
distrobox-create --root -n parrot-root --image docker.io/parrotsec/security

# Kali as root container
distrobox-create --root -n kali-root --image docker.io/kalilinux/kali-rolling
```

Enter them:
```sh
distrobox-enter -n parrot-root
# or
distrobox-enter -n kali-root
```

## Using Distrobox

**Distrobox** can save a lot of time when you need an older, modified, or stable version of a specific distro.  
You can simply create the container with the desired image and use it fully (CLI and GUI) without affecting your main system.  

If you create the container **without** `--root`, it runs normally with GUI and CLI access, but without root privileges on the host.  
This is great for isolating applications and testing safely without risking your main setup.

However, some tools — especially those that need access to low-level system resources (like raw sockets, sniffers, or certain `nmap` modes) — require elevated privileges to work properly.  
In such cases, use the `--root` flag so you can install packages and manage the container as root (using `apt`, `pacman`, etc.) **without needing sudo on the host**.

This makes Distrobox an extremely efficient and safe way to test different distros, packages, and configurations while keeping your main system clean and stable.

### Create non-root Distroboxes (user inside container — no sudo prompt inside container)
```sh
# Parrot as non-root (safe, runs as your user inside)
distrobox-create -n parrot --image docker.io/parrotsec/security

# Kali as non-root
distrobox-create -n kali --image docker.io/kalilinux/kali-rolling
```

Enter them:
```sh
distrobox-enter -n parrot
# or
distrobox-enter -n kali
```

A non-root Distrobox is handy when you want to avoid host sudo prompts and keep files owned by your user. Use the non-root one for day-to-day tooling and the `-root` one when you need to do low-level package changes or test things that require root.

### Naming convention suggestion
Do not create two containers with the same name. Use a clear suffix:
- `parrot-root` (rootfull)
- `parrot` (non-root)
- `kali-root`
- `kali`

This keeps both available and avoids collisions.

### Use container apps as native GUI apps (export)
You can export a GUI app from a Distrobox so it appears in your desktop environment / app launcher:

Example: export Burp Suite (from a Distrobox where Burp is installed)
```sh
# From inside the container or from host with -n
distrobox-export --app burpsuite -n parrot
# or
distrobox-export --app burpsuite -n kali
```

`distrobox-export --app <appname>` creates a wrapper so you can launch `burpsuite` from the host like a normal app (also works with menus/launchers).

### When you need an older version or testing without breaking host
- Use the `--root` container to safely `apt`/`pacman` downgrade/upgrade packages.
- If AUR or pacman inside a container misbehaves, use the other container (non-root or a fresh one) to continue work.

### Quick workflow examples
```sh
# 1. Create both variants
distrobox-create --root -n parrot-root --image docker.io/parrotsec/security
distrobox-create -n parrot --image docker.io/parrotsec/security

# 2. Install tools inside root container
distrobox-enter -n parrot-root
# inside container
apt update && apt install burpsuite

# 3. Export Burp to run from host
distrobox-export --app burpsuite -n parrot-root

# 4. If you prefer non-root usage later:
distrobox-enter -n parrot
# run user-level tools without host sudo
```

### Important Note About init System

You **cannot use the host systemd** inside a regular Arch host when working in Distroboxes.  
- All Distrobox containers (root or non-root) are isolated and cannot interact with the host's init system.  
- Services inside the container will only run inside the container context.  
- For anything requiring systemd on the host, you must perform it on the host itself, not inside the container.  

Everything else (apps, tools, GUIs, scripting, package management) works normally based on my experience.

### Tmux Behavior Inside Distroboxes

One very useful feature: if you use **tmux** inside a Distrobox (e.g., Kali or Parrot), all your windows and panes stay **inside the container**.  

- Splitting windows or creating new panes will **not escape** the container context.  
- This allows you to work on multiple tools or sessions **without affecting your host environment**.  
- You can have multiple tmux sessions in different containers simultaneously.  

This makes Distroboxes ideal for isolating pentesting or development workflows while keeping your main Arch system clean.


### Notes & tips
- Files created in non-root Distroboxes normally map to your home (good for editing with your host editor).
- If you want GUI integration, ensure you have X11/Wayland support and correct environment variables — `distrobox-create` usually handles this automatically.
- You can remove or recreate containers any time; keep backups of important config if you customize a container heavily.

# Done 
Enjoy your clean and powerful Arch Linux setup.
