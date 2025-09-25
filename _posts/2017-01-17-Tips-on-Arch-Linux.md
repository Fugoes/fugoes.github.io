---
layout: post
title: "Tips on Arch Linux"
date: 2017-01-17 21:58:36 +0800
tags: [Linux, Computer]
---

这篇文章主要是一些乱七八糟的使用 Arch Linux 的过程中遇到的坑和解决方法。


# 关于包管理器

使用 `yaourt -S --asdeps` 来将某一个包安装作依赖，使用这种方法的场景是安装某个包的可选依赖，因为原则上，当你卸载一个包的时候，可选依赖也应该被卸载。

另外对于Python， `pip install --user` 会将包安装在 `~/.local/` 中，这样的好处是， `pip` 安装的包不会和 `yaourt` 安装的包冲突。

# gtk 与 Qt 的和谐相处

这个可以是让 gtk 用 Qt 的主题，也可以是反过来，由于我比较亲 gnome ，所以使用的是后一种方法，首先：

```bash
yaourt -S adwaita-qt-common lxappearance
```

然后在 `qtconfig-qt4` 以及 `lxappearance` 中分别设置一下主题即可。

2017-02-10更新：

虽然我比较亲 gnome ，但是我还是用了 plasma 的主题。。。 gnome 默认主题之蠢，令人发指。。。主要的point 在于，Firefox 是用 gtk 写的(不像 chrome 是自己造了轮子)，所以 gbk 主题会影响网页的元素的效果，比如 默认主题下，网页中的按钮会特别大。。导致一些前端本意是希望并排放置的按钮变成了多行放置。。。所以：

```bash
yaourt breeze
# gtk
lxappearance
# qt4
qtconfig-qt4
# qt5
echo "export QT_STYLE_OVERRIDE=breeze" >> $HOME/.xprofile
```

然后重启或者重启一下X，世界和平！

部分 gnome 的应用在 breeze-gtk 主题下表现诡异，比如 gnome 中用来预览各种文件的 sushi ，解决方法是强行让他们用 gnome 的主题就好了，比如 sushi，修改`/usr/bin/sushi`

```bash
...
export GTK_THEME="Adwaita:dark"
...
```

世界和平！

对于有 .desktop 文件的引用只要把这句话加在 Exec 后面就好了。

# 使用休眠

首先编辑 `/etc/mkinitcpio.conf` 

```bash
HOOKS="base udev autodetect modconf block resume filesystems keyboard fsck"
```

然后执行 `mkinitcpio -p linux` 。还要为内核添加一个参数，比如我的 `swap` 分区是 `/dev/sda3` ，那么修改 `/etc/default/grub` 中的一行使之成为：

```bash
GRUB_CMDLINE_LINUX_DEFAULT="resume=/dev/sda3"
```

之后， `grub-mkconfig -o /boot/grub/grub.cfg` ，然后reboot，然后就可以使用休眠功能了。

休眠功能相当于把内存写到了 `swap` 中，写入的大小是可以控制的，当 `/sys/power/image_size` 被设置为0时，写入量最小，为了达到开机的时候就将这个值设置为0的目的，添加一个 `systemd` 的 unit ，比如 `/etc/systemd/system/limit-hibernate-image-size.service` ：

```bash
[Unit]
Description=limit hibernate image size
DefaultDependencies=no

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'tee /sys/power/image_size <<< 0'

[Install]
WantedBy=multi-user.target
```

然后 `systemd enable limit-hibernate-image-size.service` 即可。

另一个和 swap 相关的问题是系统使用 swap 的策略，也就是 `/proc/sys/vm/swappiness` ，值越大使用 swap 越激进，默认值是60，可以通过添加一个内容如下的 `/etc/sysctl.d/swappiness.conf` 来设置它的值：

```bash
vm.swappiness = 10
```

另一个值得修改的值是 `vm.min_free_kbytes` 。

# Intel 显卡

从 linux 4.8.13 升级到 4.9.6 后，屏幕出现 tearing ，解决方法是向 `/etc/X11/xorg.conf.d/20-intel.conf` 中添加 `Option "TearFree" "true"` 如下，参考 [这里](https://wiki.archlinux.org/index.php/intel_graphics#Tear-free_video) 。

```bash
Section "Device"
Identifier	"Intel Graphics"
Driver		"intel"
Option		"AccelMethod"		"sna"
Option		"TearFree"			"true"
#Option      "AccelMethod"  "uxa" # fallback
#Option      "DRI" "2"             # DRI3 is now default 
EndSection
```

# 各种软件

+ 文件管理器

  `gnome` 自带的就挺好，记得很久以前，我的一个用mac的同学说mac的文件文件管理器有一个非常好的功能，就是按空格可以预览文件，然后我在 `gnome` 默认的文件管理器里试了一些，也是可以的。。。而且这个功能看上去好几年前就有了。。。：）

  另外有一个用 Python 写的命令行的文件管理器值得推荐，就是 `ranger` ，全 `vim` 的 binding 。

+ IDE

  `jetBrains` 全家桶有学生优惠，可以下载一个 `jetbrains-toolbox` 来管理它们。

+ 浏览器

  `Firefox` 在占用资源上比 `chrome` 好多了，而且插件也更强大，同时，就在今年， `Adobe` 宣布恢复对 Linux 下的 flash 的更新，而且官方提供的 flash 性能比 chrome 中的 flash 要好一些，所以用 `Firefox` 吧！

+ 输入法

  `fcitx` + `sogou` 基本和 Windows上相同的体验，除了不能登录账号(但是 Linux 下没有乱七八糟的广告啊)，没有更多的不同了，想让它们开机启动，在 `.config/i3/config` 中添加如下内容：

  ```bash
  exec --no-startup-id "/usr/bin/fcitx"
  exec --no-startup-id "/usr/bin/sogou-qimpanel-watchdog"
  exec --no-startup-id "/usr/bin/sogou-qimpanel"
  ```

  `~/.xprofile` 中添加：

  ```bash
  export GTK_IM_MODULE=fcitx
  export QT_IM_MODULE=fcitx
  export XMODIFIERS="@im=fcitx"
  ```

  `fcitx` 还有一些很赞的小功能，比如 `Ctrl+;` 显示剪贴板历史，再比如 Unicode 输入法，在 `fcitx` 的设置里面可以设置相关的快捷键，可以用来方便地打出清真的 naïve 。

+ 壁纸

  壁纸是重要的(认真脸)，所以添加如下内容到 `.config/i3/config` ：

  ```bash
  exec --no-startup-id "/usr/bin/nitrogen --restore"
  ```

  `gnome` 自带的程序。(随口一提：`gnome` 的设置中心可以在 `i3wm` 中使用的，用来开蓝牙开声音等等还是很方便的)

+ `status bar` 小挂件

  ```bash
  yaourt -S network-manager-applet pnmixer
  ```

  `WiFi` 和设置声音大小。添加到启动项就好。

+ `mate-notification-daemon` 或 `xfce4-notifyd`  : 桌面通知
+ `rofi` : 一个和 `i3` 很配的启动器
+ `compton` : 开终端透明的需要，原则上也可以实现一些桌面特效
+ `zeal` : 用来离线阅读文档
+ `WeChat` : 对，你没看错， Linux 上有一个社区开发的基于web接口的微信
+ `netease cloud music` : 你懂的
+ `typora` : 写 markdown 必备
+ `bomi` : 一个很方便好用的播放器
+ `proxychain` : 你懂的

# 其他

+ `A stop job is running for Session c2 of user ... (1min 30s)`

    简单粗暴的方法是修改 `/etc/systemd/system.conf`

  ```bash
    DefaultTimeoutStartSec=15s
    DefaultTimeoutStopSec=15s
  ```

  有效减少了等待时间。。。

+ 添加环境变量的 ( naïve 的我以为的) 正确姿势

  ```bash
    echo "export PATH=$PATH:/home/fugoes/.local/bin" >> ~/.profile
  ```

+ evince 不能显示LaTeX生成的PDF中的中文

  安装 `poppler-data` 。
  (论用 Arch Linux 一定要看可选依赖。。。
