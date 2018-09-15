---
layout: post
title: "用Android手机播放Linux的声音"
date: 2017-02-12 10:40:51 +0800
tags: [PulseAudio, Android, Linux]
---

我的笔记本电脑的耳机口底噪非常大，大概是因为硬盘就在耳机口的位置，从 hdd 换到 ssd 之后底噪有所改善，但是还是大呀，非常影响看电影的心情，解决方案无非有：

* 接一个DAC
* 借助其他方法用安卓手机播放电脑的音频

我借过一个朋友的国砖实验， Linux PulseAudio 对国砖的 DAC 模式支持良好，但是，由于~~没有钱~~种种原因，我~~不得不~~还是尝试了一下后一种方法，效果拔群！


# 原材料

* 一台 Linux 电脑，使用 PulseAudio
* 一台有无线网卡(废话)的安卓手机

# 方案一： DLNA

[这里](https://github.com/masmu/pulseaudio-dlna)有一个项目，但是由于 DLNA 本身的限制，延时很大(秒的量级)。

# 方案二： PulseAudio over TCP

首先，打开 Google Play Store ，搜索 Simple Protocol Player ，安装之，打开，就可以看到如下~~简陋的~~界面：

![Simple Protocol Player]({{ site.url }}/assets/{{ page.title }}/picture01.jpg)

至此手机端的准备工作就做完了。

然后是电脑端的配置，参考这个软件[官方的说明](http://kaytat.com/blog/?page_id=301)：使用 `pactl list sources short` ，在列出的几项中找到类似这样的一行：

```
0	alsa_output.pci-0000_00_1b.0.analog-stereo.monitor	module-alsa-card.c	s16le 2ch 44100Hz	SUSPENDED
```

其实就是 `*.monitor` ，并记录下对应的数字，在此情况是 0 ，然后向 PulseAudio 的配置文件 `/etc/pulse/default.pa` 追加一行：

```
load-module module-simple-protocol-tcp source=0 record=true port=12345
```

`source` 参数就是前面记下的数字，然后 `pulseaudio -k &&  pulseaudio --start` 重启 PulseAudio 服务，在手机上修改 IP Address 成为你的电脑的 IP Address，然后点击播放按钮即可。

以上是官方说明里的做法，但是这样有一个问题，就是因为使用的是和电脑上的扬声器相同的输出，所以电脑上的扬声器也会有声音。。。(不过可以用这种方法来体验一下这个途径造成的延时)

解决方法也是简单的：新建一个 Virtual Output 就可以了，具体的方法参考了[这里](http://unix.stackexchange.com/questions/174379/how-can-i-create-a-virtual-output-in-pulseaudio)：

```bash
sudo modprobe snd_aloop
```

然后在音频设置页面就会多一个选项，如图所示：

![gnome setting - Sound]({{ site.url }}/assets/{{ page.title }}/picture02.jpg)

选中这个新出现的选项，然后使用 `pactl list sources short` 来查看这个 source 对应的数字，修改 PulseAudio 配置文件重启 PulseAudio 即可。

在网络情况不好(也就是路由器非常差。。。)的情况下，可以根据路由器的延时适当提高 `buffer size` 。
