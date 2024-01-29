---
title: Building WireGuard VPN for Machine Learning Server Cluster
date: 2024-01-29
lang: zh-CN
tags:
  - linux
  - network
  - wireguard
  - vpn
mathjax: true
---

## Motivation

机器学习集群需要一个安全的方式向用户暴露服务，以及跨公网服务器互联，为此需要部署 VPN 网络。

VPN 网络的部署需要考虑如下因素：

1. 网络拓扑：需要选择合适的拓扑结构以尽可能降低延迟；
2. 用户管理：可以方便地进行用户的增减和授权；
3. 使用和维护简单。

## Design

### 网络拓扑

网络拓扑决定着延迟。

延迟最低的方案显然是 full-mesh，即每一对 peer 之间都有直接的 P2P 连接。但这种拓扑结构的管理复杂度是 $\mathcal{O}(n^2)$ 的，并且每添加一个新的 peer 就需要修改所有其他 peer 的配置文件，还需要解决 NAT 带来的问题，这必须借助一些自动化的软件管理。我尝试了 [Netmaker](https://www.netmaker.io/) 和 [Headscale](https://headscale.net/)，但它们似乎都无法正确处理学校内的**复杂网络环境**，比如各种企业级路由器使用的 symmetric NAT，**成功建立 P2P 的概率非常之低**。

最终我选择了 **full-mesh 和 hub-and-spoke 相结合的拓扑**。由于服务器数量和 IP 很少变化，手动配置一个服务器间的 full-mesh 网络是可行的。与此同时，提供一个 gateway server 作为用户接入的 hub，用户只需要与 gateway server 建立连接。由于大部分用户其实是在校内使用 VPN 的，因此连接到校内的 gateway server 并转发流量并不会带来太多额外延迟。这种结构可以平衡延迟与管理复杂度，用户的增减和授权也只需要在 gateway server 上操作。

![Network Topology](topo.png)

### 协议选择

流行的 OpenVPN 和 IPSec 都足够优秀，但新兴的 WireGuard 具有无可比拟的配置简单性。对于服务端，WireGuard 可以用几行配置文件定义一个 peer 和路由；对于用户，由于 WireGuard 采用基于密钥对的认证方式，只需要一个配置文件即可接入 VPN 网络，不需要额外的密码记忆和登录操作。

### 管理方式

出于可预测性和稳定性的考量，我选择了手动配置的方法。服务器间的 full-mesh 网络一次配置后就不需要再频繁更改。而用户管理则通过一个脚本实现，当需要添加一个新用户时，脚本生成密钥对并分配 IP，把公钥和路由信息加入 gateway server 的 peer list 中，然后生成包含私钥和分配的 IP 的配置文件，并发给用户。

Gateway server 上的用户 peer 配置示例：

```ini
[Peer]
PublicKey = <redacted>
AllowedIPs = 10.1.x.y/32
AllowedIPs = fd01::x:y/128
PersistentKeepalive = 25
```

用户的接入配置文件示例：

```ini
[Interface]
PrivateKey = <redacted>
Address = 10.1.x.y/16
Address = fd01::x:y/64

[Peer]
PublicKey = <redacted>
AllowedIPs = 10.1.0.0/16  # route all VPN traffic to gateway server
AllowedIPs = fd01::/64
Endpoint = wg.ustcaigroup.xyz:51820  # gateway server is dual stack
# Endpoint = wg.ustcaigroup.xyz:51820  # IPv4
# Endpoint = wg.ustcaigroup.xyz:51820  # IPv6
PersistentKeepalive = 25
```
