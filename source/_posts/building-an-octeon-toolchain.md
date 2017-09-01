---
title: Building an Octeon toolchain
s: building-an-octeon-toolchain
date: 2017-05-19 19:11:00
tags: mips mips64 octeon edgemax edgerouter cavium octeon
---

## Prerequisites
- [Binutils 2.23.2](http://ftp.gnu.org/gnu/binutils/binutils-2.23.2.tar.bz2)
- [Cavium patches](https://dl.lochnair.net/Toolchains/Octeon/toolchain-patches-SDK-3.1.0p2-build34.tgz)
- [GCC 7.2.0](http://ftp.gnu.org/gnu/gcc/gcc-7.2.0/gcc-7.2.0.tar.xz)
- [GMP 6.1.2](http://ftp.gnu.org/gnu/gmp/gmp-6.1.2.tar.xz)
- [ISL](http://isl.gforge.inria.fr/isl-0.18.tar.xz)
- [MPC](http://ftp.gnu.org/gnu/mpc/mpc-1.0.3.tar.gz)
- [MPFR](http://ftp.gnu.org/gnu/mpfr/mpfr-3.1.5.tar.xz)
- [musl](https://www.musl-libc.org/releases/musl-1.1.16.tar.gz)

## Folder structure
Download the source archives into the `archives` folder. Extract the Cavium patchset archive into the `toolchain/patches` folder, the rest goes into the `toolchain/sources` folder.

You also need to create build/ folders for each of the components we're building (Binutils, GCC, Musl)

```
archives/
  binutils-2.23.2.tar.bz2
  ...
toolchain/
  build/
    binutils/
    gcc/
    musl/
  patches/
    binutils/
    gcc/
    ...
  sources/
    binutils-2.23.2
    ...
```

## Compiling Binutils

First we need to apply (most) of Caviums patches. Preferably we'd apply them all, but there seems to be some overlap in their patches so after 0112 patching failes. However the necessary patches for the Octeon platform works, so I didn't take the time to look more into this.

We also need a patch from GregorR's repo that enables the musl target for older Binutils versions.

```bash
cd toolchain/sources/binutils-2.23.2
for i in {0001..0112}; do
  patch -p1 -i ../../patches/binutils/$i-*
done

curl -s -L "https://github.com/GregorR/musl-cross/raw/master/patches/binutils-2.23.2-musl.diff" | patch -p1

cd -
```


Now we can compile and install Binutils. Be aware that I'm using `nproc` in the make command to determine how many processors are available. If you don't have coreutils installed you should specify the amount of jobs you want to run manually.

```bash
cd build/binutils
../../sources/binutils-2.23.2/configure --prefix=/opt/cross --target=mips64-octeon-linux-musl --disable-multilib --disable-werror
make -j$(nproc)
sudo make install
cd -
```

## GCC - First stage

We're gonna compile GCC and musl in multiple stages. First we're gonna compile the GCC compiler.

```bash
cd build/gcc
../../sources/gcc-7.1.0/configure --prefix=/opt/cross --target=mips64-octeon-linux-musl --disable-fixed-point --disable-multilib --disable-sim --enable-languages=c,c++ --with-abi=64 --with-float=soft --with-mips-plt
make -j$(nproc) all-gcc
sudo make install-gcc
```

## Kernel headers

We need the kernel headers for the next steps so enter the kernel folder and install them.

```bash
sudo make ARCH=mips INSTALL_HDR_PATH=/opt/cross/mips64-octeon-linux-musl/ headers_install
```

## musl - First stage

Here we're building some objects required by libgcc in the next step.

```bash
cd build/musl
../../sources/musl-1.1.16/configure --prefix=/opt/cross/mips64-octeon-linux-musl --host=mips64-octeon-linux-musl
make obj/crt/crt1.o
make obj/crt/mips64/crti.o
make obj/crt/mips64/crtn.o
sudo install obj/crt/crt1.o /opt/cross/mips64-octeon-linux-musl/lib
sudo install obj/crt/mips64/* /opt/cross/mips64-octeon-linux-musl/lib
sudo mips64-octeon-linux-musl-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o /opt/cross/mips64-octeon-linux-musl/lib/libc.so
sudo make install-headers
cd -
```

## GCC - Second stage

Now that we have bootstrapped musl, we can compile and install libgcc for the target.

```bash
cd build/gcc
make -j$(nproc) all-target-libgcc
sudo make install-target-libgcc
cd -
```

## musl - Second stage

With libgcc installed we can finish off musl. Note that we have to run the `./configure` script again to detect the location of libgcc.

```bash
cd build/musl
../../sources/musl-1.1.16/configure --prefix=/opt/cross/mips64-octeon-linux-musl --host=mips64-octeon-linux-musl
make -j$(nproc)
sudo make install
cd -
```

## GCC - Third stage

Lastly we'll compile the C++ standard library in GCC and install it.

```bash
cd build/gcc
make -j$(nproc)
sudo make install
```

If you've done everything correctly, you should now have a musl toolchain for the Octeon architecture.