---
layout: post
title: "Chromebook 手记"
date: 2018-09-19 18:21:47 +0800
---

暑假刚开始的时候我就种草了 Chromebook ，而且恰在公司看到有人使用三星的 Chromebook Plus ，之后又拿 [heroxbd](http://heroxbd.github.io/) 的 Chromebook Plus 玩了大半个月，终于决定购买它(然后发现汇率涨了好多)。于是我托去美国暑研的朋友帮忙购买，花去 410 刀，同时包装盒子由于不方便携带被朋友丢弃，不过据称包装里除去机器本身，仅有一个充电器和手写笔的替芯。 PD 充电器是 15V 30W 的(不要也罢)。

Chromebook Plus V1 主要的优点是：
* 续航超长(出门一天无压力，配合上一个大一些的移动电源，几乎可以连续使用 24 小时，同时它的 PD 充电支持 5V ，如果是看视频的话，它的 Chrome browser 是支持硬解的)；
* 屏幕极佳(400 nit 的 2K 屏幕，只要不到 3000 CNY)；
* 有手写笔；
* Hackable；
* 如果选择不太清真的 Chrome OS 的话，可以安装 Android APP ；
* 不贵。

主要的缺点有：
* 键盘比较糟糕(不过在长期在公司使用 MBP 2017 的键盘后，我觉得所有键盘都可以被原谅)；
* 内置的 32 GB emmc 有点不够用，扩展 SD 卡有诸多限制；
* 4 GB 内存；
* 需要科学的网路才能使用。

## Android
使用不太清真的 Chrome OS 可以直接安装 Android APP(可以直接装 apk ，也可以用 Google Play) ，比较坑的是， Android APP 只能安装在 emmc 上，而且例如网易云音乐这种 APP 居然没法把文件存到 SD 卡上，于是放弃重度使用 Android 功能的想法。

## Crouton
[crouton](https://github.com/dnschneid/crouton) 是一个基于 chroot 方案的跑 GNU Userland 的东西。为了节约宝贵的内置 emmc 空间，本人购买了一张 128 GB 的 SD 卡，此处一个坑是， Chrome OS 在锁屏时候会 umount 掉 SD 卡，解决方案是将 SD 分区，然后手动 mount 一个 ext4 分区到 `/usr/local/mnt/sdcard` 下，此时 Chrome OS 就不会自作聪明地 umount 掉它。

### 安装
首先需要开启 Developer Mode ，`Ctrl+Alt+t` 可以打开一个 terminal ， 下载好 `crouton` 文件之后：
```shell
$ shell
$ cd ~/Downloads
$ sudo sh crouton -p /usr/local/mnt/sdcard/ \
    -t cli-extra \
    -m https://mirrors.tuna.tsinghua.edu.cn/debian/ \
    -r stretch \
    -n debian
```
然后使用 `sudo /usr/local/mnt/sdcard/bin/enter-chroot -n debian` 来使用它。

### CJK 问题
虽然 Chrome OS 支持各色输入法，但是默认的 terminal 对全宽字符的处理有 bug ，导致 emacs 和 vim 之类的在有中文的情况下几乎完全无法使用。我先后尝试了以下方案：
1. 套 tmux 和 screen ：并没有什么用；
2. 使用 secure shell 扩展：它和默认的 terminal 使用了相同的前端库，于是有相同的 bug ；
3. 在安卓里使用 termux 之类的 APP ssh 回 Chrome OS ：这些 terminal APP 中，输入法都无法使用；
4. 使用 xiwi 开 xfce ：试过了 fcitx 和 ibus ，都无法正常使用，而且 xiwi 不开高分支持辣眼睛，开了高分支持就很瞎眼；
5. 使用 crostini ：用不了输入法；
6. 使用 web console ：[gotty](https://github.com/yudai/gotty) 非常科学！于是我终于有了一个可以正常使用中文的 `emacs-nox` 。。。

现在我的做法是：切到 tty ，跑一个 tmux ，其中运行各种 daemon ，比如 gotty 的 daemon ：
```shell
$ gotty -a 127.0.0.1 -p 8080 -w zsh
```
之后用 `vlock` 锁上这个 tmux ，这样就不需要在图形界面中为 chroot 留一个 tab 。

### gpg
开箱即用，各种不涉及图形的功能都正常工作。

## Crostini
目前的 beta channel 的 Chrome OS 可以使用 crostini 来运行 Linux 应用，但是这家伙实际上是个 kvm 。。。可以参考[这里](https://chromium.googlesource.com/chromiumos/platform/crosvm/)，虽然 wayland 转发看上去非常厉害，但是这家伙实在太吃内存了，而 Chrome OS 默认使用 zswap ，于是开了 crostini 之后就会疯狂压缩解压缩，非常耗电，果断放弃使用。

## gentoo prefix
gentoo prefix 在 Chrome OS 中体验应该也很科学，不过我不太想 cross build aarch64 的 prefix 。。。

## 其他
* 可以把 `http://127.0.0.1:8080` pin 到 shelf ，从而按 `Alt+<number>` 即可打开一个 terminal ；
* 各种 VPN 协议都有各种插件支持；
* 全屏之后 terminal 中的 `Ctrl-n` 之类的快捷键就不会被浏览器吃掉了；
* 可以设置比如 `Esc` 、 `Caps Lock` 等按键的功能，于是可以把 `Caps Lock` 和 `Esc` 互换位置。

最后放一张图：
![emacs-nox](/assets/{{ page.title }}/screenshot.png)

# Links
* [http://scateu.me/2016/10/09/chromebook-rocks.html](http://scateu.me/2016/10/09/chromebook-rocks.html)
