---
title: "NFS Performance Tuning"
date: 2024-02-16
updated: 2024-02-17
lang: zh-CN
tags:
  - linux
  - nfs
  - losf
---

## 前言

本文是我在实践中总结出的生产场景下 10 Gbps 网络下的 NFS 性能调优指南，特别是针对**大量小文件**（Lots of Small Files, LOSF）读写的优化。

## 调优

### 硬件

网络硬件方面，**带宽**和**延迟**两者都很重要。

要保证 NFS 的性能，高带宽网络是必要的，10 Gbps 对于生产场景来说是基础要求，更高速的 InfiniBand 或者 RoCE 网络则可按照需求和预算进行选择。

对于**大量小文件**（Lots of Small Files, LOSF）场景来说，**延迟比带宽更重要**。很多性能调优教程都忽略了这一点，只关注了连续读写的性能，即使测试了 4K 随机读写，也使用了**错误的测试方法**（下文给出了正确的测试方法）。

延迟的重要性体现在，如果程序对于小文件的访问是**内秉串行化**的，**延迟会决定串行化 IOPS 的上限**。0.1 ms 的延迟决定了串行化的 IOPS 上限是 10k，而 1 ms 的延迟对应的上限则是 1k。

内秉串行化访问的场景非常多。例如，把家目录放置于 NFS 上，oh-my-zsh 的加载、python 包的加载都是内秉串行化的。1ms 的网络延迟会让这些程序慢到不可接受（例如 `import torch` 的执行需要 30s 以上）。

使用合格的企业级交换机、恰当配置的网络拓扑，可以尽量降低延迟。同时，光模块、光转电口模块的质量也有可能极大影响延迟（我原来使用的中科光电光转电口模块会引入 0.1ms 的额外延迟，导致 IOPS 下降了 2/3）。

需要注意的是，RDMA 尽管理论上能降低延迟，但实际测试中发现 10 Gbps 以太网和 100 Gbps InfiniBand 的串行化 IOPS 差距并不大，预算有限时只使用以太网也足够。

TODO: 巨型帧

### Linux Kernel

内核网络参数需要进行调整，以适应高速网络：

```ini
# Ref: https://gist.github.com/mizanRahman/40ba603759bfb5153189ccdc9dbbd1e4

# Disable TCP slow start on idle connections
net.ipv4.tcp_slow_start_after_idle = 0

# Increase Linux autotuning TCP buffer limits
# Set max to 16MB for 1GE and 32M (33554432) or 54M (56623104) for 10GE
# Don't set tcp_mem itself! Let the kernel scale it based on RAM.
net.core.rmem_max = 56623104
net.core.wmem_max = 56623104
net.core.rmem_default = 56623104
net.core.wmem_default = 56623104
net.core.optmem_max = 40960
net.ipv4.tcp_rmem = 4096 87380 56623104
net.ipv4.tcp_wmem = 4096 65536 56623104

# TCP Congestion Control
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = cake
```

在服务端和客户端都需要应用这套设置，可以写入 `/etc/sysctl.conf` 中以持久化。

### Server Side

NFS server 的线程数可以尽量调大点，服务器负载比较高时可以提升性能，我直接设成了服务器的线程数。修改 `/etc/nfs.conf`：

```ini
[nfsd]
threads=128
```

以下几个 NFS server 参数需要调整：

- `async`：将同步 IO 操作视为异步。同步读写为主的负载可以大幅提升性能，但服务器崩溃时可能造成数据丢失，对数据完整性有极高要求的情况下不推荐使用；
- `no_subtree_check`：对性能没有大影响，但在某些情况下可以提升可靠性（同时有轻微的安全风险）。参见 [1]。

### Client Side

没有特殊的理由时应该默认使用最新的 NFSv4.2，NFSv3 使用 UDP 作为底层传输方式时，在高速网络下会因为 UDP 包序列号问题导致数据损坏，参见 [2]。

以下几个 NFS client 参数需要调整：

- `proto=rdma`：网络支持 RDMA 时设置；
- `nocto`：关闭 close-to-open 缓存一致性语义。NFS 默认行为是关闭文件时会把所有更改写回到服务器。如果对于多客户端之间的文件一致性要求比较高，不推荐使用此选项；
- `ac`：启用属性缓存（attribute caching），客户端会缓存文件属性。同样。对于数据一致性要求较高的集群，不推荐使用此选项；
- `fsc`：使用 FS-Cache 缓存数据到本地。需要同时[配置 cachefilesd](https://github.com/jnsnow/cachefilesd)。奇怪的是我在测试中并没有发现数据被缓存到本地，这可能需要进一步的探究；
- `nconnect=16`：设置 NFS client 和 server 间建立 16 条 TCP 连接。NFS client 默认只建立一条 TCP 连接，所有 RPC 复用这条连接。在某些情况下这会限制连续读写的带宽。增大 `nconnect`（最大值 16）可以解决这个问题。

特别的，`noatime` / `relatime` 的设置对于 NFS 并无影响 [3]，NFS client 始终会缓存 atime 的更改。

有些教程中会推荐修改 `rsize` 和 `wsize`，这两个值在 NFSv4.2 默认协商出的即是最大值 `1048576`，因而无需手动更改，只需检查一下是否协商正确即可。

根据 [4]，`sunrpc.tcp_max_slot_table_entries` 可能会影响性能，可以适当调大（默认 `2`）。在我的测试中，我发现当遇到千万数量级的持续小文件访问负载时，NFS 有时候会卡住。当我把这个参数调大时，此问题得以解决。设置 `/etc/modprobe.d/sunrpc.conf`：

```ini
options sunrpc tcp_slot_table_entries=16384
```

有时我会遇到 `nfsd` 占用大量 CPU 且性能急剧下降的问题，同时记录到大量 `delegreturn` RPC calls。根据 [5]，可以通过禁用 `fs.leases-enable` 解决，设置 `/etc/sysctl.conf`：

```ini
fs.leases-enable = 0
```

当 `nfsd` 因为种种原因重启后，默认会有 90s 的 grace period 用于锁恢复，这段时间内 `nfsd` 会拒绝所有 `open` 请求，在内核日志中显示：

```txt
[1073511.138061] NFSD: starting 90-second grace period (net f0000000)
```

实践中发现这段时间可以适当调小，以减少 `nfsd` 重启带来的影响。设置 `/etc/default/nfs-kernel-server`：

```bash
# Options for rpc.svcgssd.
RPCSVCGSSDOPTS="--lease-time 10 --grace-time 10"
```

## 测试

TODO

## 总结

TODO

## 参考

[1] https://man.archlinux.org/man/exports.5.en#no_subtree_check

[2] https://man.archlinux.org/man/nfs.5.en#Using_NFS_over_UDP_on_high-speed_links

[3] https://man.archlinux.org/man/nfs.5.en#File_timestamp_maintenance

[4] https://learn.microsoft.com/en-us/azure/azure-netapp-files/performance-linux-concurrency-session-slots

[5] https://docs.gitlab.com/ee/administration/nfs.html#disable-nfs-server-delegation
