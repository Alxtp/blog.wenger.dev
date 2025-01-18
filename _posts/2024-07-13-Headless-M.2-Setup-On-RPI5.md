---
title: "Headless M.2 SSD Setup On Raspberry Pi 5"
date: 2024-07-13 21:00:00 +0200 
categories: [IoT]
tags: [raspberrypi, m2ssd, nvmessd]
img_path: /IoT/
image: rpi5_setup.jpg
---

Learn how to set up your Raspberry Pi 5 to boot from a fast M.2 SSD, all without a graphical interface. Actually, not entirely without, because you will need a GUI to create a bootable SD card or USB stick on another device. But you don't need to connect your Raspberry to a monitor. This step-by-step guide covers everything from hardware setup to software configuration for a headless setup on your Pi.
![RPI5 Setup](rpi5_setup.jpg)
_My personal Raspberry Pi 5 Setup with the official cooler and M.2 HAT from Waveshare_

You will need:
- Another computer to prepare the boot image and SSH into the Pi
- Micro SD card or USB stick for initial boot
- A Raspberry Pi 5 + M.2 HAT with an installed NVMe SSD

## Create Customized OS Image
The first step is to download the latest version from the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) and run the installer. Once installed:
1. Chose Raspberry Pi Device
2. Operating System -> As we make a headless install choose `Raspberry Pi OS Lite (64-bit)`
3. Storage -> Your SD card or USB

When prompted to use OS customization, say "Yes" and complete the entire configuration:
![OS Customisation](os_customisation.png)

> If you have no available WLAN you'll need to connect it to your LAN by cable.
{: .prompt-info}

Under Services, enable SSH and either create a password or generate an SSH key:
![OS Customisation](os_customisation02.png)

After your image has been created, open the drive and copy the file `firstrun.sh`{: .filepath} to your PC as you'll use it later.

## Prepare Raspberry Pi
Now you can insert the boot media into your Raspberry Pi. Note that your M.2 hat should not be connected yet, as this could cause your Pi not to boot (was the case on my site).

After a few minutes your Pi should be ready and you should be able to connect to it via SSH with `ssh -i .ssh/key_name username@hostname.local` or just `ssh username@hostname.local` if you are using password authentication.

The first thing to check is the bootorder by running `sudo rpi-eeprom-config`:
![Boot Order](bootorder.png)

The desired value should be `0xf641`, which means the following order when searching for a bootable drive:
1. SD card
2. USB disk
3. NVMe SSD

You can edit this value by running `bootorder sudo -E rpi-eeprom-config –edit`

Now that the bootorder is adjusted to our needs, you can power off the Pi by running `sudo poweroff` and disconnect it from the power source to connect your M.2 HAT. Then turn it back on and connect again using SSH.

Next, check the name of your M.2 SSD by running `lsblk -p`
![List Disks](list_disk.png)
You can see that it is called `nvme0n1`

Now create a file named `firstrun.sh`{: .filepath} by running `nano firstrun.sh`, copy the content of the `firstrun.sh`{: .filepath} file you copied earlier, and save it.

## OS installation on NVMe SSD

Now that we have prepared the Raspberry Pi and connected the NVMe SSD, we're ready to install the operating system directly onto the NVMe drive. This process involves using the Raspberry Pi Imager tool on the Raspberry Pi itself. Follow these steps:
1. `sudo apt update`
2. `sudo apt install rpi-imager`
3. `sudo rpi-imager --cli --first-run-script firstrun.sh https://downloads.raspberrypi.org/raspios_lite_arm64_latest /dev/nvme0n1`

Let's break down the last command:

- `--cli`: Use the command-line interface
- `--first-run-script firstrun.sh`: Specify the firstrun script we created earlier
- `https://...`: The URL for the latest Raspberry Pi OS Lite (64-bit) image
- `/dev/nvme0n1`: The device path for your NVMe SSD (adjust if necessary)

This command will download the latest Raspberry Pi OS Lite (64-bit) image and install it directly onto your NVMe SSD. It will also incorporate the firstrun.sh script we created earlier, which will run on the first boot of the newly installed system.

Now you can power down the Pi by running `sudo poweroff` and disconnecting your boot media. Then you can power it up again and after a few minutes you can connect to it via SSH in the same way as before!

And there you go, you have successfully installed the headless Raspberry Pi OS on a M.2 SSD without even connecting your PI to anything other than power.