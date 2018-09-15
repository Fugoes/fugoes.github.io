---
layout: post
title: "Examples on LXC and KVM"
date: 2018-04-07 14:00:00 +0800
categories: Computer
---

Recently I gave a talk on introduction to GNU/Linux, and I needed to provide each learner with a standalone GNU/Linux environment. So I created a KVM based virtual machine and ran tens of LXCs in it. This post notes down some commands. Using docker might be a better solution. However, I need to pay for the networking traffic consumption to download docker images, because my school haven't provide mirror(or proxy) for docker images.

Create the KVM image:
```bash
qemu-img create -f qcow2 debian.qcow2 32G
qemu-img create -f qcow2 lxc.qcow2 32G
```
Allocate the tap interface:
```bash
ip tuntap add dev tap0 mode tap user "$(whoami)"
```
If you want to do headless installation, you need to manually pass `console=ttyS0` to kernel:
```bash
qemu-system-x86_64 \
    -kernel vmlinuz \
    -initrd initrd.gz \
    -hda debian.qcow2 \
    -cdrom debian-9.4.0-amd64-DVD-1.iso \
    -append "console=ttyS0" \
    -nographic \
    -enable-kvm \
    -smp 12 \
    -m $((1024 * 32)) \
    -netdev tap,ifname=tap0,id=tapnet \
    -device e1000,netdev=tapnet
```
The `vmlinuz` and `initrd.gz` could be downloaded from debian's mirrors, for example: https://mirrors.tuna.tsinghua.edu.cn/debian/dists/stable/main/installer-amd64/current/images/hd-media/ . After the installation, run the KVM with:
```bash
qemu-system-x86_64 \
    -hda debian.qcow2 \
    -hdb lxc.qcow2 \
    -nographic \
    -enable-kvm \
    -smp 12 \
    -m $((1024 * 32)) \
    -netdev tap,ifname=tap0,id=tapnet \
    -device e1000,netdev=tapnet
```
It should not need root permission, since the interface is allocated in advance. Now login to the KVM. You need to pass `console=ttyS0` to kernel to use tty by mounting the KVM image and editing `/boot/grub/grub.cfg`. Then setup LXC base image as `/var/lib/lxc/debian/` using `lxc-create`. You could create some simple scripts to create tens of overlay LXC instances. Here is an example configuration for overlay LXC images:
```bash
lxc.network.type = veth
lxc.network.flags = up
lxc.network.link = lxcbr0

lxc.include = /usr/share/lxc/config/debian.common.conf

lxc.tty = 4
lxc.arch = amd64

lxc.rootfs = aufs:/var/lib/lxc/debian/rootfs:/var/lib/lxc/debian-01/rootfs
lxc.utsname = debian-01
lxc.network.hwaddr = 00:FF:AA:00:00:01
```
Install `dnsmasq` and `aufs` in KVM (don't forget to stop the `dnsmasq.service`, or the `dnsmasq` instance started by `lxc-net` would refuse to start), and doing some NAT, so that these LXCs could gain networking access. Note that `aufs` is not a block level copy-on-write file system, it may cause huge space usage if modifying huge files in the overlay LXC container.

To make a `qcow2` file smaller in file size:
```bash
qemu-img convert -O qcow2 -c lxc.qcow2{,_smaller}
```
To attach a `qcow2` file:
```bash
qemu-nbd -c /dev/nbd0 debian.qcow2
```
To detach it:
```bash
qemu-nbd -d /dev/nbd0
```
