---
layout: post
title: "Arch Linux Rocks"
date: 2017-01-17 20:45:33 +0800
tags: [Linux, Computer]
---

[Arch Linux](https://www.archlinux.org/) 是茫茫多的 `Linux` 发行版之一，按照官方的 `wiki` ，这是一个追求[简单、现代、实用主义、用户中心](https://wiki.archlinux.org/index.php/Arch_Linux)的发行版。个人觉得在实用主义以及用户中心这两点上， Arch Linux 当之无愧。

这篇文章内容包括安装基本系统以及图形界面。


# 安装基本系统

## 联网

Arch Linux 的安装重度依赖网络，所以首先需要联网，以无线网络为例，首先使用 `ip link` 查看 `interface` ，比如 `wlp7s0` ，使用 `wifi-menu -o wlp7s0` 来连接网络。

## 磁盘准备

我使用 `btrfs` 作为文件系统，由于 `btrfs` 不(直接)支持 `swapfile` ，所以需要三个分区，分别用作 `swap` 、 `esp` 分区以及 `btrfs` 的分区，使用 `parted /dev/sda` 进行分区，常用命令输入 `help` 即可。最后得到在 `parted` 中 `print` 的结果：

```bash
Model: ATA SanDisk SDSSDA24 (scsi)
Disk /dev/sda: 240GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system     Name  Flags
 1      1049kB  52.4MB  51.4MB  fat32                 boot, esp
 2      52.4MB  231GB   231GB   btrfs
 3      231GB   240GB   9202MB  linux-swap(v1)
```

格式化各个分区：

```bash
mkfs.vfat -F 32 /dev/sda1
mkfs.btrfs /dev/sda2
```

在 `btrfs` 的分区中新建几个 `subvolume` ：

```bash
mount /dev/sda2 /mnt
cd /mnt
btrfs subvolume create @arch
btrfs subvolume create @boot
btrfs subvolume create @home
```

然后 `umount /mnt` 再挂载相关分区：

```bash
mount -o subvol=@arch /dev/sda2 /mnt
```

## 安装基础软件

编辑镜像列表 `/etc/pacman.d/mirrorlist` 反注释喜欢的镜像即可。安装基本的软件包：

```bash
pacstrap -i /mnt base base-devel
```

挂载相关分区：

```bash
mount -o subvol=@boot /mnt/boot
mount -o subvol=@home /mnt/home
mkdir -p /mnt/boot/esp
mount /dev/sda1 /mnt/boot/esp
```

以这样的方式挂载的好处是，除了 `grub` 之外，所有的文件都在 `btrfs` 的分区里，官方的推荐是将 `esp` 分区挂在 `/boot` ，会导致 `grub` 的文件以及内核等文件都在 `esp` 中，当你有多系统的需求时可能会不够灵活，并且另一个好处是防手残，因为 `esp` 分区完全可以不挂载。

## 编辑分区信息

可以根据 `/etc/mtab` 来写 `/etc/fstab` ，官方推荐的脚本 `genfstab` 从某个时候开始就有 bug ，所以手动写更科学。

## `chroot`

```bash
arch-chroot /mnt /bin/bash
```

## 各种设置

```
pacman -S neovim
nvim /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
tzselect
```

## 安装引导器

```bash
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/esp --bootloader-id=grub2 --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```

## 其他软件的安装

```bash
pacman -R netctl
pacman -S networkmanager btrfs-progs
systemctl enable networkmanager
```

## 收尾的工作

```bash
mkinitcpio -p linux
exit
umount -R /mnt
reboot
```

## 重启之后

```bash
hostnamectl set-hostname dell
```

`systemd` 会自动新建一个 `subvolume` ，路径为 `mkdir /var/lib/machines` ，可以删掉之后新建一个同样路径的文件夹。

# 安装图形界面

## 装X

```bash
pacman -S xorg xorg-xinit xterm
```

可以 `startx` 一下看看是否正常。

## `yaourt`

在 `/etc/pacman.conf` 追加：

```
[archlinuxcn]
SigLevel = Optional TrustAll
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

然后：

```bash
pacman -Syu archlinuxcn-keyring yaourt
```

即可使用 `yaourt` 来替代 `pacman` 了。

## 添加用户

先用 `passwd` 设置 root 用户的密码，然后：

```bash
useradd -m fugoes
```

添加用户，使用 `gpasswd -a fugoes audio` 将用户添加到 `audio` 组中，其他的组可以参考 `/etc/group` 文件。

## 安装一个桌面环境

比如 `gnome` ：

```bash
yaourt -S gnome
```

这样的好处是大多数常用软件都会被装上．

## 安装 `i3wm`

```bash
yaourt -S i3
yaourt -S lightdm-gtk-greeter
systemctl enable lightdm
systemctl start lightdm
```

## 各种图形驱动

```bash
yaourt xf86-video
```

## bumblebee

```
yaourt -S bumblebee bbswitch
```

一定要严格按照 [官方维基](https://wiki.archlinux.org/index.php/Bumblebee) ，这是一个 work out of box 的软件，至少在我的机器上。

对于不需要开图形但是需要GPU的场景， `echo ON > /proc/acpi/bbswitch` 即可。
