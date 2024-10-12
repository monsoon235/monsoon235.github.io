---
title: "关于 IPv6 地址分配的一切"
date: 2024-10-12
lang: zh-CN
tags:
  - network
  - router
  - ipv6
  - slaac
  - dhcp
  - dhcpv6
  - dhcpv6-pd
---

## 序言

IPv4 只有一种动态地址分配方式，即 DHCP，但 IPv6 就有 SLAAC 和 DHCPv6 两种分配方式，同时 DHCPv6 还存在 PD (Prefix Delegation) 的扩展。这三种分配方式之间又存在交互，使得 IPv6 分配过程中出现的问题远比 IPv4 多。大多数可以搜到的教程只从表面解决了问题，对于其后的技术细节模棱两可，而没有从根本上厘清 IPv6 与 IPv4 的差异，

此文旨在从相关基础概念出发，授人以渔地讲清楚 IPv6 三种地址分配方式的工作原理，帮助彻底解决 IPv6 分配中的疑难杂症。

## IPv6 基础概念

### LLA (Link-Local Address，链路本地地址) 和 EUI-64

LLA 其实在 IPv4 中就已存在，当 DHCP 没有正常工作时，一些操作系统就会为网络接口分配一个 `169.254.0.0/16` 的地址，用于临时的点对点通信。但 LLA 在 IPv4 中并不重要，只扮演一个可有可无的备用角色，只有当 DHCP 故障时才会出现，因而绝大部分人（包括笔者）直到 IPv6 普及时才了解到 LLA 的存在。

IPv6 LLA (`fe80::/8`) 继承了 IPv4 LLA 点对点通信的基本功能，但更进一步承担了 NDP (Neighbor Discovery Protocol，邻居发现协议) 以及 SLAAC (Stateless Address Autoconfiguration，无状态地址自动配置) 的重要功能。理解它才能理解 SLAAC 的工作原理。

举例来说，当两个网口通过网线直接相连后，就会分别自动生成 IPv6 LLA，如 `fe80::dfc2:d2aa:c86f:171e/64` 和 `fe80::da8f:9d5b:57e3:c6a6/64`，两者都可以 `ping` 通对方的 LLA。在 Linux 上通过 `ip -6 route` 命令，可以查到自动配置的 LLA 路由项：

```txt
fe80::/64 dev eth0 proto kernel metric 1024 pref medium
```

IPv6 LLA 使用特定的算法从 MAC 地址中生成，即 EUI-64，例如网口的 MAC 地址为 `70:07:12:34:56:78` 时，生成的 EUI-64 为 `7207:12ff:fe34:5678`，LLA 则为 `fe80:7207:12ff:fe34:5678/64`（EUI-64 加上 `fe80` 的前缀）。具体的生成方式下图所示：

![IPv6 LLA 生成过程，图源 https://www.networkacademy.io/ccna/ipv6/stateless-address-autoconfiguration-slaac](generating-link-local-address-example.png)

一般而言，路由器不会转发 LLA 地址的流量，它**仅用于链路点对点通信**。

### GUA (Global Unicast Address，全局单播地址)

IPv6 GUA (`2000::/3`) 可以对应到 IPv4 “公网 IP”的概念。理论上它是全球唯一的，并且可以用于公网通信。一个配置良好的网络架构应当能使每个设备都获取到 IPv6 GUA，以最大程度上发挥 IPv6 的 P2P 通信优势。

### 私有地址

`fc00::/7` 被定义为 IPv6 的私有地址，类似于 IPv4 中 的 `10.0.0.0/8`、`172.16.0.0/12` 和 `192.168.0.0/16`，用于局域网通信。与 LLA 不同的是，它可以被路由器转发。

由于 IPv6 被设计为全球每个设备都能分到 GUA，私有地址在 IPv6 中的作用被大大削弱。当无法做到为每个设备分配 GUA 时（如一些校园网环境），在内网分配 IPv6 私有地址可以作为替代方案，让内网设备可以访问 IPv6。

### 组播 (Multicast)

IPv6 组播地址（`ff00::/8`）与 IPv4 组播地址（`224.0.0.0/4`）类似，用于网段内的一对多通信。**SLAAC 和 DHCPv6 都依赖组播工作**。常用的组播地址有：

- `ff02::1`：本地链路所有节点；
- `ff02::2`：本地链路所有路由器。

### NDP (Neighbor Discovery Protocol，邻居发现协议)

NDP 工作于 ICMPv6 之上，类似于 IPv4 ARP，用于发现数据链路层中其他节点和相应的 IPv6 地址，并确定可用路由和维护关于可用路径和其他活动节点的信息可达性。**SLAAC 基于 NDP 工作**，涉及的报文类型有：

1. RS（Router Solicitation）和 RA（Router Advertisement）：用于配置 IPv6 地址及路由；
2. NS（Neighbor Solicitation）和 NA（Neighbor Advertisement）：用于查找链路上其他设备的 MAC 地址。

## SLAAC (Stateless Address Autoconfiguration, 无状态地址自动配置)

SLAAC 是 [RFC 4862](https://datatracker.ietf.org/doc/html/rfc4862) 中定义的 IPv6 地址分配方式，也是**推荐的分配方式**。事实上 Android 只支持 SLAAC IPv6 分配。

SLAAC 最大的特点就是无状态（stateless），即不需要一个中心化的服务器来负责分配。下面笔者用一个例子说面 SLAAC 的过程。

假设**路由器**上的 `lan0` 网口和**主机**上的 `eth0` 网口相连，`lan0` 的 LLA 是 `fe80::1/64`，`eth0` 的 MAC 地址为 `70:07:12:34:56:78`。同时，路由器持有 `2001:db8::/64` 的 GUA 前缀，即这个子网下所有 GUA 都会被上级路由器路由到此路由器的 `wan` 网口。SLAAC 的流程如下：

1. `eth0` 根据 MAC 地址生成 EUI-64 `7207:12ff:fe34:5678` 和 LLA `fe80:7207:12ff:fe34:5678/64`；
2. 主机执行 DAD（Duplicated Address Detection）确保 LLA 在本地链路中唯一。其和地址分配无关，因而在此略过，有兴趣的读者可以自行查阅相关资料；
3. 主机通过 `eth0` LLA 发送 RS 消息。RS 使用组播地址 `ff02::2` 发送给本地链路所有的路由器。
4. 路由器回复 RA 消息给 `eth0` LLA。RA 中包含前缀 `2001:db8::/64`、有效期和 MTU 等信息。
5. 主机收到 RA，将前缀和 EUI-64 组合成 `2001:db8::7207:12ff:fe34:5678/64` 分配给 `eth0`，并添加路由表：

   ```txt
   2001:db8::/64 dev eth0 proto ra metric 1024 expires 2591993sec pref medium
   default via fe80::1 dev eth0 proto static metric 1024 onlink pref medium
   ```

6. 主机进行 DAD 检测，并使用 NA 消息向链路上的邻居通告新地址的使用。

![SLAAC 流程，图源 https://www.networkacademy.io/ccna/ipv6/stateless-address-autoconfiguration-slaac](ipv6-stateless-address-autoconfiguration.gif)

SLAAC 看起来很美好，但有个**重要缺陷**：不支持 DNS 信息的下发，主机必须通过其他方式（通常是 DHCPv6）获取 DNS。RA 中有两个标志位用以解决此问题：

- `M` (Managed Address Configuration)：可以通过 DHCPv6 获取地址信息；
- `O` (Other Configuration)：可以通过 DHCPv6 获取其他信息（如 DNS）。

而更新的 [RFC 6106](https://datatracker.ietf.org/doc/html/rfc8106) 则通过在 RA 中添加 RDNSS（Recursive DNS Server）和 DNSSL（DNS Search List），支持了 DNS 信息的下发。各操作系统对于 RDNSS 的支持度见 [Comparison of IPv6 support in operating systems](https://en.wikipedia.org/wiki/Comparison_of_IPv6_support_in_operating_systems)。在实际使用中，绝大部分情况下只需要配置 IPv4 DNS（通过 DHCPv4 获得），因而 RDNSS 扩展的意义并不大。

以上基于 EUI-64 的 SLAAC 地址配置存在的问题是，**它生成的地址是固定并且可预测的**，这会带来安全性和隐私问题。[RFC 4941](https://datatracker.ietf.org/doc/html/rfc4941) 定义的 IPv6 SLAAC 隐私扩展解决了这一问题。它在 SLAAC 时同时生成随机的、定期更换的地址，以解决隐私问题。同时 EUI-64 生成的地址也被保留，用于外部传入连接。在启用隐私扩展的情况下，Linux 中生成的 IPv6 地址例如（从上到下分别是隐私地址、EUI-64 GUA、LLA）：

```txt
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc cake state UP group default qlen 1000
    link/ether 70:07:12:34:56:78 brd ff:ff:ff:ff:ff:ff
    inet6 2001:db8::dead:beef:aaaa:bbbb/64 scope global temporary dynamic
       valid_lft 2591998sec preferred_lft 604798sec
    inet6 2001:db8::7207:12ff:fe34:5678/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 2591998sec preferred_lft 604798sec
    inet6 fe80:7207:12ff:fe34:5678/64 scope link
       valid_lft forever preferred_lft forever
```

## DHCPv6

DHCPv6 和 DHCPv4 的运行方式整体相同，主机发送 `ff02::1:2` UDP 端口 547 的组播消息，DHCPv6 server 回复地址和 DNS 等信息。

有所不同的是，DHCPv6 可以在有状态或者无状态的模式下运行，两者的区别在于是否获取地址。当搭配 SLAAC 使用时，主机只需要从 DHCPv6 获取 DNS 等信息，因而可以使用无状态 DHCPv6。

## DHCPv6 PD (Prefix Delegation, 前缀委托)

PD 是 [RFC 3633](https://datatracker.ietf.org/doc/html/rfc3633) 定义的 DHCPv6 扩展。它用于在网络中分发 IPv6 前缀。

在启用 PD 扩展的情况下，DHCP server 向主机发送一个 IPv6 子网前缀（如 `2001:db8::/56`）的使用权，并添加路由表以确保将此子网下的地址全部路由到请求前缀的主机。主机可以再对此子网进行划分和分配。

一个典型的 DHCPv6 PD 使用场景是家庭 ISP 网络接入。家庭网关路由器向 ISP DHCP server 请求 IPv6 前缀，然后再通过 SLAAC 在家庭内网中分发此前缀子网中的地址。

## 总结

本文简要介绍了 IPv6 地址分配中涉及的一些概念，并阐述了 SLAAC、DHCPv6、DHCPv6 PD 的工作原理。在简化地址管理这一方面，IPv6 可以说做得并不成功，多种标准并存，且存在不同的组合形式，让客户端会有不小的概率无法正确获取 IPv6。

在实际情况中，我们最常预见的 IPv6 分配情况有三种：

- 纯 SLAAC：一般校园网（教育网）属于此类。在实际使用中，笔者发现存在错误配置的内网主机胡乱发送 RA 的情况，导致整个内网所有主机的 IPv6 都错误配置。与此同时，在这种模式下，自行接入的路由器将无法再向下级设备分发 SLAAC GUA，因为 SLAAC 基于的本地链路组播数据包无法被路由器转发（可以通过 IPv6 桥接或者 NAT6 解决，此处不展开说明）。
- 纯 DHCPv6：一些企业内网会使用此模式，因为 DHCPv6 可以集中管理。这种模式最大的问题是 [Android 不支持 DHCPv6](https://www.nullzero.co.uk/android-does-not-support-dhcpv6-and-google-wont-fix-that/)。但在其他操作系统下，此模式运行较为稳定。
- SLAAC + DHCPv6 PD：这是家庭 ISP 网络接入最常见的模式，大部分家用路由器都对此做了适配，可以做到开箱即用。

## 参考

- [IPv6 Stateless Address Auto-configuration (SLAAC)](https://www.networkacademy.io/ccna/ipv6/stateless-address-autoconfiguration-slaac)
- [RFC 4862: IPv6 Stateless Address Autoconfiguration](https://datatracker.ietf.org/doc/html/rfc4862)
- [RFC 6106: IPv6 Router Advertisement Options for DNS Configuration](https://datatracker.ietf.org/doc/html/rfc8106)
- [RFC 4914: Privacy Extensions for Stateless Address Autoconfiguration in IPv6](https://datatracker.ietf.org/doc/html/rfc4941)
- [RFC 3633: IPv6 Prefix Options for Dynamic Host Configuration Protocol (DHCP) version 6](https://datatracker.ietf.org/doc/html/rfc3633)
- [Android does not support DHCPv6 and Google 'Won't Fix' that](https://www.nullzero.co.uk/android-does-not-support-dhcpv6-and-google-wont-fix-that/)
- [Comparison of IPv6 support in operating systems](https://en.wikipedia.org/wiki/Comparison_of_IPv6_support_in_operating_systems)
