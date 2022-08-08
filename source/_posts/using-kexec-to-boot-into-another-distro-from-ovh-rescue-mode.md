---
title: Using kexec to boot into another distro from OVH rescue mode
date: 2022-08-08 17:08:49
tags:
---
## Motivation

I recently acquired a dedicated server from OVH of the cheaper variety. This means it does not include any IPMI solution I could make use of to get a console to install my preferred distro.

So since there's sometimes quirks with using the vendor's rescue I figured I'd try to somehow boot into a "live CD" from the rescue system.

My first attempt at this was making my own Arch Linux ISO with a password set and the bootloader changed to always copy-to-RAM, then writing that with dd to the disk on the server.

This worked, but it's only viable if there's nothing on the server, if you have an installation you don't wanna overwrite, it gets more complicated.

Thus - enter kexec. This allows us to start another kernel from our existing one, and the rescue system kernel does have this kernel option enabled.

## Prerequisites
* Another server somewhere with a HTTP server installed
* An Arch Linux install to make the ISO on (other distros might be fine too, but pacman is required)

## Making the ISO
Install the `archiso` package with pacman:

```bash
pacman -S archiso
```

Create a folder to work in and copy the profile:

```bash
mkdir archiso
cd archiso
cp -r /usr/share/archiso/configs/releng profile
cd profile
```

Generate a password hash with openssl:

```
openssl passwd -6
Password:
Verifying - Password:
$6$c9Itar6beCUka9gC$OjZwx888i166Oi6naDn3wrF316Em2nOtsypxEukaNasGdi4x6dCvsjO8MS/e/q0rjmXJ5fJdX3sG0n0d92Dmc.
```

Open `airootfs/etc/shadow` with your preferred editor, and add the hash so it looks like this:

```
roota few seconds have access:$6$c9Itar6beCUka9gC$OjZwx888i166Oi6naDn3wrF316Em2nOtsypxEukaNasGdi4x6dCvsjO8MS/e/q0rjmXJ5fJdX3sG0n0d92Dmc.:14871::::::
```

For completeness what I did to change the bootloader to copy the live CD image to RAM was modify the `syslinux/archiso_sys-linux.cfg` file, so the first APPEND line looked like this (i.e. exactly the same as the copy-to-RAM boot option):

```
APPEND archisobasedir=%INSTALL_DIR% archisolabel=%ARCHISO_LABEL% copytoram
```

Now you're ready to build the ISO, jump out of the profile folder and start the build:
```bash
cd ..
sudo mkarchiso -o out -w work profile
```

Depending on the connection on the machine you're this on, this might take a while, grab a cup of coffee!

When it's done, upload the ISO to the server you're hosting the HTTP server on.

## Making the files available
Now you're ready to extract the built ISO on the webserver you prepared.
Personally I used Nginx with the default configuration on an Ubuntu install.

So I simply did:
```bash
sudo apt install nginx
sudo mount -o loop ~/archlinux-2022.08.08-x86_64.iso /mnt
sudo cp -rv /mnt/* /var/www/html/
sudo umount /mnt
```

You'll likely have to adapt this somewhat for your configuration if you're using another distro or web server.

## Booting into Arch
Then it's time to boot your dedicated server into rescue mode and prepare to start Arch.

Install the userspace part of kexec (if it asks about handling reboots, you can just select no):
```bash
apt install kexec-tools
```

Download the kernel and initramfs from the server you set up:
```bash
wget http://<SERVER_IP>/arch/boot/x86_64/vmlinuz-linux
wget http://<SERVER_IP>/arch/boot/x86_64/initramfs-linux.img
```

Load the new kernel and initramfs into the running kernel:
```bash
kexec -l vmlinuz-linux --initrd=initramfs-linux.img --command-line="archisobasedir=arch archiso_http_srv=http://<server_ip>/ ip=dhcp"
```

The command line arguments lets the initramfs know where to find the files, and that it should use DHCP to get the network configuration. More information on how to configure this flag can be found [here](https://wiki.archlinux.org/title/mkinitcpio#Using_net).

Then it's time to boot into the Arch environment:
```
systemctl kexec
```

Note: `kexec -e` can also be used here, but it won't handle unmounting filesystems first.

Hopefully you should in a few seconds have access to an Arch live environment with the password you set.

## References
* https://wiki.archlinux.org/title/archiso
* https://wiki.archlinux.org/title/Preboot_Execution_Environment#HTTP
* https://wiki.archlinux.org/title/Kexec