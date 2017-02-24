---
title: Creating a Wheezy container with LXD
s: wheezy-container-with-lxd
date: 2017-01-05 12:46:53
tags:
---
As most of those who've read my blog posts before, I'm providing sch_cake binaries for the EdgeRouter series. Currently whenever there's an update to the source I manually compile it for all the models.

I've been planning to set up a build server for this for a while now, but I've met some roadblocks while doing so. One of them being the Wheezy-image for LXD provided by LinuxContainers not being able to resolve DNS. After a little investigation I found out that the resolvconf package needed to automatically configure DNS resolvers properly isn't installed by default.

Since you can't use apt-get to install it when you don't have DNS working, you can download it to your host and push it to the container using LXD's file command. As soon it's available inside the container you can install it using dpkg.

```bash
wget "http://ftp.de.debian.org/debian/pool/main/r/resolvconf/resolvconf_1.67_all.deb"
lxc file push resolvconf_1.67_all.deb wheezy/root/
lxc exec wheezy -- dpkg -i resolvconf_1.67_all.deb
```

As soon as the resolvconf package is installed you can configure your resolvers as you usually do.