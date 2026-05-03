---
title: "Prepare Ubuntu Servers with Subiquity Autoinstall"
date: 2026-05-03 17:00:00 +0200 
categories: [Self-Hosting]
tags: [ubuntu, automation]
media_subpath: /selfhost/
---

Recently, I was searching for a simple way to prepare the nodes for my self-hosted Kubernetes cluster. For my cluster, I bought 3 refurbished client machines, and my ultimate goal was to just insert a USB, boot it, and then connect via SSH without requiring a monitor or keyboard attached to the clients. It also needed to prepare the client for Ansible, which would deploy the cluster.

I looked into different possibilities like Debian preseeding, Talos Linux, or PXE boot, which all didn't seem optimal to me. I wanted a simpler solution and was too inexperienced to use something like Talos Linux. After some research, I found Subiquity Autoinstall (aka Ubuntu Autoinstall), which seemed just right.

It uses the same YAML syntax as cloud-init, but instead of running at first boot, Autoinstall runs during installation and lets you answer installation configuration questions like timezone, user, and keyboard beforehand, thus completely automating the installation of Ubuntu.

## Prepare Autoinstall Config

The first step is to create the YAML configuration that specifies the required settings. All settings not mentioned will use their default values. Reference here: [https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html).

For my cluster, I used the following setup. Most of the keys are self-explanatory; the others are explained with comments.

```yaml
autoinstall:
  version: 1
  user-data:
    users:
      - name: luke
        lock_passwd: true #disable password
        groups: adm, plugdev, sudo
        shell: /bin/bash #when not specified /bin/sh will be used
        sudo: ALL=(ALL) NOPASSWD:ALL #skip password prompt when using sudo
        ssh_authorized_keys:
          - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGWC5r4Hsbey06TsO0pGoQ+Egy9Ctn8kq6ZOduVpImhz luke
  keyboard:
    layout: ch
  source:
    id: ubuntu-server-minimal #install only essential packages
  apt:
    mirror-selection:
      primary:
        - uri: "https://ubuntu.ethz.ch/ubuntu"
        - uri: "https://mirror.init7.net/ubuntu"
  packages:
    - avahi-daemon #will be used after installation to discover the IP (DNS Service Discovery)
  ssh:
    install-server: true
    allow-pw: false #disable password auth for SSH
  timezone: "Europe/Zurich"
  updates: all 
  late-commands:
    - echo "x-wing-$(cat /target/etc/machine-id | cut -c1-4)" > /target/etc/hostname #set unique hostname (eg. x-wing-12bh6)
  shutdown: poweroff #shutdown machine after installation
```
{: file="autoinstall.yaml" }

## Create Installer ISO

With the `autoinstall.yaml` ready, the next step is to download the official Ubuntu Server ISO and mount it.

```bash
wget "https://releases.ubuntu.com/24.04.3/ubuntu-24.04.3-live-server-amd64.iso"
```

When mounted, you should find and copy the file under `boot/grub/grub.cfg`{: .filepath}. This file specifies the GRUB boot selection and initially looks like this:

```
set timeout=30

loadfont unicode

set menu_color_normal=white/black
set menu_color_highlight=black/light-gray

menuentry "Try or Install Ubuntu Server" {
	set gfxpayload=keep
	linux	/casper/vmlinuz  ---
	initrd	/casper/initrd
}
menuentry "Ubuntu Server with the HWE kernel" {
	set gfxpayload=keep
	linux	/casper/hwe-vmlinuz  ---
	initrd	/casper/hwe-initrd
}
grub_platform
if [ "$grub_platform" = "efi" ]; then
menuentry 'Boot from next volume' {
	exit 1
}
menuentry 'UEFI Firmware Settings' {
	fwsetup
}
else
menuentry 'Test memory' {
	linux16 /boot/memtest86+x64.bin
}
fi
```
{: file="boot/grub/grub.cfg" }

The Autoinstall process will need to be manually confirmed at the beginning, unless we create a new menu entry or modify the first menu entry as follows. Also, the timeout can be reduced so the installation starts quicker:

```
set timeout=5

...

menuentry "Autoinstall Ubuntu Server" {
	set gfxpayload=keep
	linux	/casper/vmlinuz autoinstall quiet  ---
	initrd	/casper/initrd
}
```
{: file="grub.cfg" }

Now, the last step is to pack everything we created together into a new ISO:

```bash
xorriso -indev ubuntu-24.04.3-live-server-amd64.iso \
        -outdev autoinstall-ubuntu-server-24.04.3.iso \
        -map autoinstall.yaml /autoinstall.yaml \
        -update grub.cfg /boot/grub/grub.cfg \
        -boot_image any replay
```

And thats it. Now this ISO can be burned to a USB and inserted in your device. When theres no other boot device configured, it should boot from the USB, install Ubuntu Server according to your wishes and shut down at the end. Start it again and connect with SSH.

---

![Avahi Logo](avahi-logo.png){: width="400"}
_Avahi Logo_

Ahh, almost forgot: With the package `avahi-daemon` that I installed in my example, the host can be conveniently discovered over the network with `nmap`:

```bash
nmap -sn 192.168.1.0/24
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-03 17:02 +0200
Nmap scan report for x-wing-7d3d.local (192.168.1.63)
Host is up (0.00016s latency).
```
