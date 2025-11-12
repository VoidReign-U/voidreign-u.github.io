---
title: Virtual Machines + Containers Setup Guide
author: VoidReign-U
date: 2025-11-12 00:00:00 +0800
categories:
  - Linux
tags:
  - System
permalink: /System/qvc
---


# Virtual Machines + Containers Setup Guide

## Void Linux

### Install Required Packages

```bash
xbps-install dbus qemu libvirtd virt-manager
```

If not already installed, install network packages:

```bash
xbps-install bridge-utils iptables2
```

### Enable Services

```bash
ln -s /etc/sv/dbus /var/service
ln -s /etc/sv/libvirtd /var/service
ln -s /etc/sv/virtlockd /var/service
ln -s /etc/sv/virtlogd /var/service
```

### VirtualBox 

```bash
xbps-install virtualbox virtualbox-ose-additions virtualbox-ose-dkms
```

### Containers: Podman + Distrobox

```bash
xbps-install podman podman-compose podman-docker distrobox
```

### Docker

```bash
sudo xbps-install -S docker
sudo ln -s /etc/sv/docker /var/service/
sudo sv up docker
docker --version
```

## Arch Linux

### 1. Install QEMU/KVM Dependencies

```bash
sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat ebtables nftables libguestfs
```

> Tip: I recommend using `qemu-full`. If prompted to remove `iptables`, select **Yes**. `nftables` replaces `iptables`.  
> **Important:** Update your system before installing:

```bash
sudo pacman -Syu
```

### 2. Configure Libvirt

Edit `/etc/libvirt/libvirtd.conf`:

```ini
unix_sock_group = "libvirt"
unix_sock_rw_perms = "0770"
```

Add your user to the `libvirt` group and activate it:

```bash
sudo usermod -a -G libvirt $(whoami)
newgrp libvirt
```

Enable and start the libvirt service:

```bash
sudo systemctl enable --now libvirtd
```

### 3. Install VirtualBox (Optional)

```bash
sudo pacman -S virtualbox virtualbox-guest-iso virtualbox-host-dkms
```

>  Reboot after installation, then launch `virt-manager`.

### 4. Containers: Podman + Distrobox

```bash
sudo pacman -S podman podman-compose podman-docker distrobox
```

### 5. Docker

```bash
sudo pacman -S docker docker-compose distrobox
```

---

## Debian-based Linux Distributions

### 1. Oracle VirtualBox Setup

Add repository to `/etc/apt/sources.list` (replace `<mydist>` accordingly):

```text
deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] https://download.virtualbox.org/virtualbox/debian <mydist> contrib
```

Download and register Oracle public key:

```bash
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | \
sudo gpg --yes --output /usr/share/keyrings/oracle-vbox-2016.gpg --dearmor
```

Install VirtualBox:

```bash
sudo apt-get update
sudo apt-get install virtualbox-7.1
```

> Replace `virtualbox-7.1` with `virtualbox-7.0` if desired.

### 2. Check Virtualization Support

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

> Output > 0 means virtualization is enabled. If 0 → enable **VT-x** (Intel) or **AMD-V** (AMD) in BIOS.

### 3. Install QEMU & Virt-Manager

```bash
sudo apt install qemu-kvm qemu-system qemu-utils python3 python3-pip \
libvirt-clients libvirt-daemon-system bridge-utils virtinst libvirt-daemon virt-manager -y
```

Verify `libvirtd` service:

```bash
sudo systemctl status libvirtd.service
```

### 4. Configure Default Network

Start and enable default network:

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

Check status:

```bash
sudo virsh net-list --all
```

Expected output:

```
 Name      State      Autostart   Persistent
----------------------------------------------
 default   active     yes         yes
```

### 5. Add User to libvirt

```bash
sudo usermod -aG libvirt $USER
sudo usermod -aG libvirt-qemu $USER
sudo usermod -aG kvm $USER
sudo usermod -aG input $USER
sudo usermod -aG disk $USER
```

> Reboot to apply changes.

### 6. Containers Setup

**Podman + Distrobox:**

```bash
sudo apt-get install podman podman-compose podman-docker distrobox
```

**Docker (from Docker repo, not Debian repo):**

```bash
sudo apt-get install docker.io distrobox
```

> Note:
> 
> - If using `ufw` or `firewalld`, Docker-exposed ports bypass firewall rules.
>     
> - Docker works with `iptables-nft` or `iptables-legacy`. `nft` firewall rules are not supported. Ensure rules are added to the `DOCKER-USER` chain.
>     

### 7. Docker Engine Installation on Debian

Supported versions:

- Debian 13 Trixie (stable)
    
- Debian 12 Bookworm (oldstable)
    
- Debian 11 Bullseye (oldoldstable)
    

**Add Docker’s GPG key:**

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

```


**To install the latest version, run:**

```console
 sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**The Docker service starts automatically after installation. To verify that Docker is running, use:**

```console
 sudo systemctl status docker
```

**Some systems may have this behavior disabled and will require a manual start:**

```console
 sudo systemctl start docker
```


## Fedora && RHEL

Install mandatory virtualization packages:

```bash
sudo dnf install @virtualization
```

Optional full install:

```bash
sudo dnf group install --with-optional virtualization
```

Enable and start libvirtd:

```bash
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```

Add user to libvirt group:

```bash
sudo usermod -a -G libvirt $(whoami)
```

### Podman

```bash
sudo dnf -y install podman
```

For `podman machine` commands:

```bash
sudo dnf -y install podman-machine
```

### Distrobox

Enable COPR repository:

```bash
sudo dnf copr enable alciregi/distrobox
```

Install Distrobox:

```bash
sudo dnf install distrobox
```

### Docker

```bash
sudo dnf install toolbox
```
