---
title: Adding Cake support on the EdgeRouter Lite
s: cake-support-on-edgerouter-lite
date: 2016-09-18 13:53:00
tags:
---
**Update (9/Oct/2016):** The NAT feature in Cake now supports older kernels.

**Update (6/Oct/2016):** NAT support has entered the main Cake repository. The feature currently only works on newer versions of the Linux kernel, so in my fork I've added an option to disable it, which is used below. 

**Update (5/Oct/2016):** I've found another way to cross-compile for the MIPS architecture, and have updated the tutorial accordingly.

For a while I've been wanting to try out Cake on my ERL. It was pretty difficult to get it to work, but it was well worth it.

This tutorial is divided into two parts, one for compiling the sch_cake module and one for compiling iproute2.

Compile the sch_cake module for the ERL
---------------------------------------

### Prerequisites
- Debian Wheezy machine
- Git
- Cavium SDK (get it from [cnusers](http://www.cnusers.org/index.php?option=com_remository&Itemid=32&func=fileinfo&id=184) (requires registration), or [my mirror](http://static.lochnair.net/bufferbloat/cnusers_sdk_3.1.0.tgz))
- [ERL 1.9 firmware GPL archive](http://dl.ubnt.com/firmwares/edgemax/v1.9.0/GPL.ER-e100.v1.9.0.4901118.tbz2)

Download and extract the Cavium SDK (currently v3.1):
```bash
mkdir edgeos; cd edgeos
wget "http://static.lochnair.net/bufferbloat/cnusers_sdk_3.1.0.tgz"
tar xzvf cnusers_sdk_3.1.0.tgz
```

Setup your environment to use the Octeon SDK
```bash
cd OCTEON-SDK
source ./env-setup OCTEON_CN50XX
cd ..
```

Download and extract the UBNT GPL archive:
```bash
wget "http://dl.ubnt.com/firmwares/edgemax/v1.9.0/GPL.ER-e100.v1.9.0.4901118.tbz2"
tar xjvf GPL.ER-e100.v1.9.0.4901118.tbz2
```

Extract the Cavium Executive headers:
```bash
tar xzvf source/cavm-executive_4899453-g82e0782.tgz
```

Create the correct folder structure and extract the kernel:
```bash
mkdir -p linux-src/linux-3.10
cd linux-src/linux-3.10
tar xzvf ../../source/kernel_4899453-gc1c94e2.tgz
cd kernel
```

Prepare and compile the kernel (replace -j2 with how many CPU cores you have in your system):
```bash
make ARCH=mips CROSS_COMPILE=mips64-octeon-linux-gnu- prepare
make ARCH=mips CROSS_COMPILE=mips64-octeon-linux-gnu- -j2
```

When the kernel is done compiling return to the edgeos folder and clone my sch\_cake repository. It contains a fix to make sch\_cake compile on this kernel.

```bash
git clone https://github.com/Lochnair/sch_cake.git
cd sch_cake
```

Compile the module:
```bash
make ARCH=mips CROSS_COMPILE=mips64-octeon-linux-gnu- KDIR=../linux-src/linux-3.10/kernel/
```

Upload the resulting sch\_cake.ko file to your ERL and copy it to your /lib/modules/$(uname -r)/kernel/net/sched folder.

Lastly run modprobe to load our new module.
```bash
sudo depmod -a
sudo modprobe sch_cake
```

Compile iproute2 with cake support
----------------------------------

I've covered how to set up a build environment for MIPS cross-compiling in another post [here](https://www.lochnair.net/setting-up-build-environment-for-mips-el/). Follow that guide, then come back here.

When you've got a MIPS machine running Debian Wheezy up, install the needed packages to build iproute2.

```bash
apt-get install build-essential flex git bison iptables-dev
```

Clone my git repository containing the modified iproute2-4.4 source. This repository contains the UBNT iproute2 source modified with cake support from Dave Taht's [tc-adv](https://github.com/dtaht/tc-adv) repository.

```bash
git clone https://github.com/Lochnair/iproute2-4.4-cake-ubnt.git
```

Enter the iproute2 folder and run make.
```bash
cd iproute2-4.4-cake-ubnt
make
```

When it's done copy the resulting ip/ip and tc/tc files from the iproute2 folder and replace the ones on the ERL with your new ones.

Congratulations, you now have Cake support on your ERL!

Of course, this won't work with the Vyatta configuration at the moment, so you'll have to configure Cake manually using tc. There's a good start configuration at the [Cake wiki](https://www.bufferbloat.net/projects/codel/wiki/Cake/#configuring-cake).