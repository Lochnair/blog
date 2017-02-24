---
title: Setting up a build environment for MIPS(el)
s: build-environment-for-mips
date: 2016-10-05 19:26:00
tags:
---
When compiling userspace tools for the EdgeRouter models (I'm using the same method when compiling the kernel for the ER-X), I've been using the QEMU emulator to emulate the MIPS architecture, as opposed to typical cross-compiling. But it's a pain to set up, and in my case it's been horribly slow to run.

Recently I found a tutorial over at the [Debian wiki](https://wiki.debian.org/EmDebian/CrossDebootstrap#QEMU.2Fdebootstrap_approach) that shows how to set up a debootstrap environment for another architecture. ARM is used in that particular example, but the same methods apply for MIPS too.

Please note, this method still uses QEMU to emulate the MIPS architecture, but it does so transparently when chrooting into the debootstrapped environment, so it's a lot easier to use.

Note: Remember to replace mips with mipsel if your platform is using little endian.


First install QEMU and debootstrap:
```bash
sudo apt install binfmt-support qemu qemu-user-static debootstrap
```

Now we'll bootstrap the new system:
```bash
sudo debootstrap --foreign --arch mips wheezy wheezy-mips
```

The QEMU emulator for the target architecture needs to be available in the bootstrapped system for chrooting to work:
```bash
sudo cp -v /usr/bin/qemu-mips-static wheezy-mips/usr/bin/
```

Run second stage of debootstrap:
```bash
sudo DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true \
 LC_ALL=C LANGUAGE=C LANG=C chroot wheezy-mips /debootstrap/debootstrap --second-stage
```

Finally trigger post install scripts:
```bash
sudo DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true \
 LC_ALL=C LANGUAGE=C LANG=C chroot wheezy-mips dpkg --configure -a
```

Now it's time to chroot into our new environment:
```bash
sudo chroot wheezy-mips /bin/bash
```

When I started using my new environment I quickly noticed that the locales weren't properly set up, so I got a lot of warnings, especially when running apt-get. This is easily fixed by installing the `locales` package and then configuring it with the locales you need.

```bash
apt-get install locales
dpkg-reconfigure locales
```