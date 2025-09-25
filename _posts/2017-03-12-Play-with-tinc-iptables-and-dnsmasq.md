---
layout: post
title: "Play with tinc, iptables and dnsmasq"
date: 2017-03-12 15:30:34 +0800
tags: [network, linux, tinc, iptables, dnsmasq, ipset]
---

看到 bigeagle 的[这篇博客](https://bigeagle.me/2016/02/ipset-policy-routing/)，感觉非常棒，遂决定试一试，中途遇到了一些坑，好在都顺利解决了，遂记录之。(本文基于 Linux )

# 目标

```
        +----+
   +----| do |----+
   |    +----+    |
   |              |
   |              |
   |              |
   |              |
+--+---+     +----+----+
| dell |     | tencent |
+------+     +---------+
```

如图所示， do 是一台有公网 IP 的 VPS ， tencent 是一台在(玄学) NAT 里的 VPS， dell 是我的笔记本，目标是所有机器都连接到 do ，并且 dell 可以直接访问(也就是不走 do ，直接穿透 NAT ) tencent 。

# Step 1: Setup Tinc-VPN

可以参考的[一篇博客](https://cn2.chionlab.moe/2016/12/12/better-way-to-bypass-gfw-with-tinc/)的 `tinc 安装及配置` 部分。

# Step 2: Setup Tinc as Gateway

tinc 配置如下：

```
        +----+                   do: Address = 192.168.100.1
   +----| do |----+
   |    +----+    |
   |              |            dell: Address = 192.168.100.2
   |              |
   |              |
   |              |         tencent: Address = 192.168.100.3
+--+---+     +----+----+
| dell |     | tencent |
+------+     +---------+
```

此时互相 ping Address 都可以 ping 通。

## Server Side Setup

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -j MASQUERADE -o eth0
```

至此， do 就可以作为 Gateway 来使用了。

## Client Side Setup

官方有一个[栗子](http://tinc-vpn.org/examples/redirect-gateway/)，这个栗子其实已经基本满足需求了，但是有一个缺点就是需要手动设置一下 VPS 的路由，必须让目的地是 VPS 的包走外网而不是 tinc 的虚拟的 interface (我的 tinc 对应的 interface 是 flux)，如果你用作 gateway 的那台机器在NAT里面，而且NAT最外面的公网 IP 还会变化，这种做法就变得不现实，在折腾了一番之后，我找到了一个有效的解决方案。具体的做法是，新建一个用户给 tinc 使用(比如 `sudo useradd tinc -s /bin/nologin` )，然后在 iptables 中通过 match uid 来实现给 tinc 用户的包打上 MARK ，再通过 ip rule 为这些包单独设置一个路由表。这样 tinc 就会处理好 NAT 以及 IP 的变化。

中途遇到的一个坑是， tinc 本身提供了一个 `--user=` 的选项，但是使用这个选项指定用户之后并不是预期的效果，发往 VPS 的公网 IP 的包会走错 interface ，我猜测的原因是， tinc 启动时会首先使用 root 权限新建一个 tun interface ， 导致实际上的需要发往 VPS 公网 IP 的包的 uid 是 0 (也就是 root ) ，导致上面的做法失效，解决方法是，手动新建一个 uid 是 tinc 的 tun interface ：

```bash
sudo ip tuntap add dev flux mode tun user tinc group tinc
sudo mknod /dev/net/flux c 10 200
sudo chown tinc:tinc /dev/net/flux
```

然后：

```bash
sudo bash -c "echo '' > /etc/tinc/flux/tinc-up"
sudo bash -c "echo '' > /etc/tinc/flux/tinc-down"
```

注意修改一下 `/etc/tinc/flux/tinc.conf` ：

```
...
Device = /dev/net/flux
Interface = flux
...
```

运行 tinc 时，使用 `sudo -u tinc tincd blablabla` ，然后一切就和预期的相同了。

具体的脚本见后文。

# Step 3: Setup Dnsmasq & ipset

这一部分主要参考了[ bigeagle 的博客](https://bigeagle.me/2016/02/ipset-policy-routing/)，不多赘述，最后综合上一步的方法以及大鹰的博客得到的一份脚本如下：

```bash
$ cat ~/bin/tinc
#!/bin/bash
dev="wlp7s0"
tinc_gateway="192.168.100.1"
tinc_user="tinc"
tinc_group="tinc"
INTERFACE="flux"

up() {
    if [ -a /dev/net/$INTERFACE ]; then
        echo "File exist!"
    else
        echo "Create a nod!"
        sudo mknod /dev/net/$INTERFACE c 10 200
        sudo chown $tinc_user:$tinc_group /dev/net/$INTERFACE
    fi

    sudo ip tuntap add dev $INTERFACE mode tun user $tinc_user group $tinc_group

    uid=$(id -u $tinc_user)

    sudo iptables -t mangle -N DIRECT
    sudo iptables -t mangle -A DIRECT -m owner --uid-owner $uid -j MARK --set-mark 42
    sudo iptables -t mangle -A DIRECT -m set --match-set direct dst -j MARK --set-mark 42

    sudo iptables -t mangle -A OUTPUT -j DIRECT
    sudo iptables -t mangle -A POSTROUTING -j DIRECT

    sudo iptables -t nat -A POSTROUTING -o $dev -m mark --mark 42 -j MASQUERADE

    ip route show dev $dev | while read gwroute; do
        sudo ip route add $gwroute dev $dev table direct
    done

    sudo ip rule add fwmark 42 table direct

    sudo ip link set $INTERFACE up
    sudo ip addr add 192.168.100.2/32 dev $INTERFACE
    sudo ip route add 192.168.100.0/24 dev $INTERFACE

    # A small trick to preserve the default route
    sudo ip route add 0.0.0.0/1 dev $INTERFACE
    sudo ip route add 128.0.0.0/1 dev $INTERFACE

    sudo mkdir -p /var/run/tinc
    sudo chown -R $tinc_user:$tinc_group /var/run/tinc

    sudo -u $tinc_user tincd -n flux -D --debug=3 --pidfile=/var/run/tinc/$INTERFACE.pid
}

down() {
    sudo kill $(cat /var/run/tinc/$INTERFACE.pid)
    sudo rm -rf /var/run/tinc

    sudo ip route del 0.0.0.0/1 dev $INTERFACE
    sudo ip route del 128.0.0.0/1 dev $INTERFACE

    sudo ip route del 192.168.100.0/24 dev $INTERFACE
    sudo ip addr del 192.168.100.2/32 dev $INTERFACE
    sudo ip link set $INTERFACE down

    sudo ip rule del table direct
    sudo ip route flush table direct

    sudo iptables -t nat -F
    sudo iptables -t mangle -F
    sudo iptables -t mangle -X DIRECT

    sudo ip tuntap del dev $INTERFACE mode tun

    sudo rm /dev/net/$INTERFACE
}


case $1 in
    up )
        up ;;
    down )
        down ;;
esac
```

将这个脚本丢到 PATH 里，然后使用 `tinc up` 启动代理，`tinc down` 关闭之。

可以添加到 systemd 开机启动：

```bash
$ cat tinc-flux.service
[Unit]
Description=Tinc!
After=multi-user.target

[Service]
Type=simple
User=fugoes
ExecStart=/home/fugoes/bin/tinc up
ExecStop=/home/fugoes/bin/tinc down

[Install]
WantedBy=multi-user.target
```

注意到脚本中给为 direct table 配置的路由表是根据当前的默认路由生成的，所以换一个网络环境需要重启一下服务。

另外还有一个小 Tip 是：如果 ipset 不设置 timeout ， 那个 table 就会越来越大，这当然是不科学的，所以修改一下 ipset 的配置：

```bash
$ cat /etc/ipset.conf
create direct hash:ip family inet hashsize 1024 maxelem 65536 timeout 14400
add direct 166.111.8.28 timeout 0
```

对于需要永久有效的 ipset 项目，设置 timeout 为 0 即可。

但是这样做会带来另一个问题，如果你的 timeout 设置的过小，小于某个 DNS 结果的 ttl ，那么 ipset 中的对应条目会在 timeout 秒之后失效，这个时候，由于 DNS 的 ttl 还没有结束，所以 dnsmasq 不会重新查询这个条目，于是 ipset 中就会有一段时间没有这个条目，解决方案也是简单的，配置一下 dnsmasq 即可：

```bash
sudo bash -c 'echo "max-cache-ttl=10800" >> /etc/dnsmasq.conf'
```

只要保证这个 max-cache-ttl 的值大于 ipset 的 timeout ，在至多 max-cache-ttl 时间之后， dnsmasq 会更新一次 ipset 中对应的条目，该条目的 timeout 会被重置为默认值 (在我的栗子中是 14400 )。

PS. 本文中的服务端使用的是 debian testing ，客户端使用的是 Arch Linux ，不同发行版配置文件位置可能不同。每台电脑的网卡对应的 interface 名字不一定相同，自行修改。
