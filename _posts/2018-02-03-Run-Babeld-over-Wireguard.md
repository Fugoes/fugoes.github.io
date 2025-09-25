---
layout: post
title: "Run Babeld over Wireguard"
date: 2018-02-03 11:38:00 +0800
categories: Computer Network
---

[Babeld](https://github.com/jech/babeld) is a loop-avoiding distance-vector routing protocol. It could auto update link cost according to network delay, which is different from other distance-vector routing protocols like RIP. This unique feature makes it useful for personal overlay network. [Wireguard](https://www.wireguard.com/) is a simple and fast layer 3 VPN software.

Wireguard doesn't support IPv6 multicast, which is used by Babeld to do neighbor discovery. Wireguard forwards packets by matching `allowed-ips` option for its node. For a central node which other nodes connect to, you could decide which peer to forward a packet to by including the packet's destination IPv6 address in the peer's `allowed-ips`. These `allowed-ips` works like some kind of routing table. However, `allowed-ips` for different peers cannot overlap, which means you can not assign one address to multiple peers' `allowed-ips`. As a result, you could only assign the multicast IPv6 address to one peer, and other peers would never receive `Babeld Hello` message from this central node. So you could only use Wireguard as a point-to-point layer 3 tunnel to run Babeld over it, or Babeld would fail to discover all links. As a result, for a n nodes full mesh network, each nodes need a Wireguard tunnel to all other n - 1 nodes.

By default, Wireguard would not assign IPv6 link local address to interface. In fact, Wireguard's interface doesn't have MAC address since it is an layer 3 VPN software. However, Babeld requires a IPv6 link local address to work. I use the following code to generate random IPv6 link local address for such interface to make Babeld happy:

```python
import random

def random_mac():
    digits = [0x00, 0x16, 0x3e, random.randint(0x00, 0x7f), random.randint(0x00, 0xff), random.randint(0x00, 0xff)]
    return ":".join(map(lambda x: "%02x" % x, digits))

def mac_to_ipv6(mac):
    parts = mac.split(":")
    parts.insert(3, "ff")
    parts.insert(4, "fe")
    parts[0] = "%x" % (int(parts[0], 16) ^ 2)
    ipv6_parts = []
    for i in range(0, len(parts), 2):
        ipv6_parts.append("".join(parts[i:i + 2]))
    return "fe80::%s/64" % (":".join(ipv6_parts))

def random_ipv6():
    return mac_to_ipv6(random_mac())
    
if __name__ == "__main__":
    print(random_ipv6())
```

Then just add these random generated IPv6 link local address to all interface(as well as normal IPv4 addresses) and set both ends of a Wireguard tunnel's `allowed-ips` to `0.0.0.0/0,::/0`, which simply allow everything to go through the tunnel. Finally, start Babeld on all the nodes and all the interface with your configuration, it shall work.
