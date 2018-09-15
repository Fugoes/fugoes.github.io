---
layout: post
title: "Manual Bootstrap Gentoo Prefix for aarch64 Android"
date: 2018-02-01 22:18:00 +0800
categories: Gentoo
---

## Overview

There is a helper script to aid bootstrapping Gentoo Prefix on different platforms. Please browse [this wiki page](https://wiki.gentoo.org/wiki/Project:Prefix) to get this script. This `bootstrap-prefix.sh` works for arm64 platform with some manual tuning, due to the absent of `arm64/prefix` profile. After [this commit](https://gitweb.gentoo.org/repo/gentoo.git/commit/?id=4979fb71526b7919d218dededaf1c8803d201573), it shall work smoothly. This script divides the whole process into three stages, namely stage1, stage2, and stage3. Stage1 and stage2 will bring you a tool chain in `${EPREFIX}/tmp`, as well as a `portage` and its dependencies. Stage3 will using the tool chain in `${EPREFIX}/tmp` to bootstrap a complete system.

## Preparation

You need an native tool chain as well as other programs like `bash` to run the script. There are ways to achieve these requirements.

### Method 0: Using an aarch64 Gentoo prefix

Yes, Gentoo prefix on Linux(RAP) could bootstrap itself to a different prefix path. This method is less time consuming than the following two, since you could directly go to stage3 without doing stage1 & 2 with an existing Gentoo prefix installed.

### Method 1: Using a native GNU/Linux environment

You could choose any Linux Distro that support aarch64 and install it on some device(Raspberry Pi 3 is a option).

The problem with this method is that it is hard to find powerful aarch64 machine running GNU/Linux. Using `chroot` with an Android device, or `qemu-chroot` with an amd64 GNU/Linux are acceptable workaround. After `chroot` into an aarch64 GNU/Linux, you could easily obtain a working tool chain with the package manager. Note that, `qemu-chroot`'s performance is not so good, though still acceptable. Generally speaking, a normal laptop's performance under `qemu-chroot` is worse than a recent Android phone. What's more, `qemu-chroot` doesn't support all system calls, I encountered some errors when trying to bootstrap a stage3 Gentoo prefix with `qemu-chroot`, while stage1 & stage2 compiles happily with `qemu-chroot`, and you could copy stage2 to you Android device and doing stage3 there.

### Method 2: Cross compile an aarch64 tool chain

There is an outdated [wiki](https://wiki.gentoo.org/wiki/Project:Android/build). Note that, current cross `emerge` command ignores `EPREFIX` in its environment, but offers an command line options `--prefix` for it. I haven't succeeded with this method.

## Stage1 & 2

Before compiling everything, you might want to read [this wiki page](https://wiki.gentoo.org/wiki/Project:Android/tarball).

According to [this wiki page](https://wiki.gentoo.org/wiki/Project:Prefix/Manual_Bootstrap)(It worths reading):

```bash
$ export EPREFIX="/data/gentoo-arm64"
$ export PATH="${EPREFIX}/usr/bin:${EPREFIX}/bin"
$ export PATH="${PATH}:${EPREFIX}/tmp/usr/bin:${EPREFIX}/tmp/bin:/usr/bin:/bin"
$ ./bootstrap-prefix.sh "${EPREFIX}" setup
$ ./bootstrap-prefix.sh "${EPREFIX}" stage1
$ ./bootstrap-prefix.sh "${EPREFIX}" stage2
```

If you are lucky enough, you shall run these commands without any errors. If you encounter some errors, try to fix it by reading build logs. You don't need to delete all files between two trail, the script will handle it correctly. You might want to do a snapshot between commands as checkpoint or change portage configuration during the process.

Some of the problems I encountered are:

1. Wrong `gid` for `/dev/pts` on Android and `glibc` refuse to compile, to fix it:

   ```bash
   $ /system/bin/mount -t devpts -o remount,gid=5 devpts /dev/pts
   ```

2. `/bin/bash` not found on Android(or something like that), you could fix it by doing a simple soft linking.

3. Some package failed to merge since trying to installed files outside of `$EPREFIX/`. I encountered this problem when forcing a stable keywords by adding `ACCEPT_KEYWORDS="arm64 -~arm64"` to `make.conf`. To fix it:

   ```bash
   $ echo "sys-devel/gcc-config ~arm64" >> "${EPREFIX}/etc/portage/package.accept_keywords"
   ```

   Or you could just stick to the default unstable keywords which works fine.

4. `groupadd` and `useradd` not found, to workaround it:

   ```bash
   $ touch "${EPREFIX}/bin/groupadd" && chmod +x "${EPREFIX}/bin/groupadd"
   $ touch "${EPREFIX}/bin/useradd" && chmod +x "${EPREFIX}/bin/useradd"
   ```

   Then manually editing `/etc/{shadow,group}` if necessary.

5. `emerge` failed to download distfiles(e.g. due to `wget` not found), just download it else where and put it under portage's `distfile/` directory.

## Stage3

If you use Method 0 and you still want to use the helper script to do stage3, you need to manually do necessary soft links under `/bin` and `${EPREFIX}/{bin,usr/bin}` for bash, python, etc. Then soft link the `usr/portage` directory in previous Gentoo prefix to the new one. Don't forget to include the previous Gentoo prefix in `PATH`:

```bash
$ export PATH="${EPREFIX}/usr/bin:${EPREFIX}/bin"
$ export PATH="${PATH}:${EPREFIX}/tmp/usr/bin:${EPREFIX}/tmp/bin"
$ export PATH="${PATH}:${PREVIOUS_EPREFIX}/usr/bin:${PREVIOUS_EPREFIX}/bin"
```

Note that, the sequence matters.

After copying stage2 to your Android device, you could use the helper script to bootstrap stage3:

```bash
$ ./bootstrap-prefix.sh "${EPREFIX}" stage3
```

You might encounter errors similar to what I listed in previous section, and the solutions are similar, too. Other than already listed errors, problems I encountered in this stage are:

1. Failed to build `gmp` due to some C++ errors. Simply disable the `cxx` use flag for `gmp`, this use flag is not needed to build `gcc`.
2. Wrong shebang, fix it by manual editing the shebang or doing soft linking.

## Finishing

You shall do a `emerge -e @system` after finishing doing stage3. You could use `distcc` to make it less time consuming. And since you might encounter errors, while `emerge -e` will not remember which package you have already rebuild:

```bash
$ emerge -e -pv @system > pkgs
```

Then edit `./pkgs` to get a package list. Do not delete packages' version! Then:

```bash
$ cat ./pkgs|while read pkg; do
	emerge --oneshot --nodeps "${pkg}" || exit 1
done
```

## Using the prefix

### Script to start prefix

You might want to use a script to start the prefix. It is a bit more tricky for Android. Please refer to [this wiki page](https://wiki.gentoo.org/wiki/Project:Android/tarball) for more information.

### SSH

The `etc/init.d/sshd` script doesn't work well with prefix. You might want to run `sshd` manually.

```bash
HOME=/data/gentoo-arm64/root /data/gentoo-arm64/usr/sbin/sshd -E sshd.log -p 22222
```

Put your public key in `/data/gentoo-arm64/root/.ssh/authorized_key` and change sshd's configuration so that root user could login. If you are still not able to do ssh, try adding `StrictModes No` to the configuration file and restart the daemon.

### Distcc

Distcc is a tool to distribute C family language compilation to speed things up. A general guide could be found [here](https://wiki.gentoo.org/wiki/Distcc). For this scenery, cross Distcc is needed. [This](https://wiki.gentoo.org/wiki/Distcc/Cross-Compiling) is a guide for cross Distcc. I encountered some __problems__ with the cross Distcc. The `${EPREFIX}/usr/lib/distcc/bin}` folder shall be `${EPREFIX}/usr/lib64/distcc/bin` for aarch64 devices.

Distcc's plain mode would run the preprocessing on the local machine and the pump mode would try to run preprocessing remotely. Sometimes distcc pump mode might failed to build some package(because pump mode assumes headers will not change during the compilation, but for some package, e.g. `llvm`, this is not true), try using Distcc's plain mode instead. You could still enable the `lzo` option with plain mode, which would reduce the the networking load.

