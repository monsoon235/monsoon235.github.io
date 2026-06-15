---
title: NFS Performance Tuning
date: 2024-02-16
lastmod: 2024-02-17
slug: 2024-02-16-nfs-tuning
tags:
  - linux
  - nfs
  - losf
---

## Introduction

This article is a guide to NFS performance tuning over a 10 Gbps network in production scenarios, which I have distilled from practice. It focuses in particular on optimizing the reading and writing of **Lots of Small Files** (LOSF).

## Tuning

### Hardware

On the network hardware side, both **bandwidth** and **latency** matter.

To guarantee NFS performance, a high-bandwidth network is necessary. 10 Gbps is the baseline requirement for production scenarios; faster InfiniBand or RoCE networks can be chosen according to your needs and budget.

For the **Lots of Small Files** (LOSF) scenario, **latency is more important than bandwidth**. Many tuning tutorials overlook this and focus only on sequential read/write performance; even when they test 4K random read/write, they use the **wrong testing method** (the correct method is given below).

The importance of latency lies in the fact that if a program's access to small files is **intrinsically serialized**, **latency determines the upper bound of serialized IOPS**. A latency of 0.1 ms caps serialized IOPS at 10k, while a latency of 1 ms corresponds to a cap of 1k.

Intrinsically serialized access scenarios are very common. For example, when the home directory is placed on NFS, the loading of oh-my-zsh and the loading of Python packages are both intrinsically serialized. A 1 ms network latency makes these programs unacceptably slow (e.g., executing `import torch` takes more than 30s).

Using a decent enterprise-grade switch and a properly configured network topology can minimize latency as much as possible. At the same time, the quality of optical modules and optical-to-electrical port modules can also have a huge impact on latency (the Chinet (中科光电) optical-to-electrical port module I originally used introduced an extra 0.1 ms of latency, causing IOPS to drop by 2/3).

It should be noted that although RDMA can theoretically reduce latency, in actual testing I found that the difference in serialized IOPS between 10 Gbps Ethernet and 100 Gbps InfiniBand is not large; when the budget is limited, using only Ethernet is sufficient.

TODO: jumbo frames

### Linux Kernel

The kernel network parameters need to be adjusted to suit a high-speed network:

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

This set of settings needs to be applied on both the server and the client; it can be written into `/etc/sysctl.conf` to make it persistent.

### Server Side

The number of NFS server threads can be set as large as possible; it can improve performance when the server load is relatively high, and I simply set it to the number of threads on the server. Modify `/etc/nfs.conf`:

```ini
[nfsd]
threads=128
```

The following NFS server parameters need to be adjusted:

- `async`: treats synchronous I/O operations as asynchronous. For workloads dominated by synchronous reads/writes this can greatly improve performance, but it may cause data loss when the server crashes; it is not recommended when there are extremely high requirements for data integrity;
- `no_subtree_check`: has no major impact on performance, but in some cases it can improve reliability (with a slight security risk at the same time). See [1].

### Client Side

When there is no special reason, you should use the latest NFSv4.2 by default. When NFSv3 uses UDP as the underlying transport, it can cause data corruption over high-speed networks due to UDP packet sequence number issues; see [2].

The following NFS client parameters need to be adjusted:

- `proto=rdma`: set when the network supports RDMA;
- `nocto`: disables close-to-open cache consistency semantics. The default NFS behavior is to write all changes back to the server when a file is closed. If you have relatively high requirements for file consistency across multiple clients, this option is not recommended;
- `ac`: enables attribute caching, so the client caches file attributes. Likewise, for clusters with high requirements for data consistency, this option is not recommended;
- `fsc`: uses FS-Cache to cache data locally. You also need to [configure cachefilesd](https://github.com/jnsnow/cachefilesd). Strangely, in my testing I did not find data being cached locally; this may require further investigation;
- `nconnect=16`: sets up 16 TCP connections between the NFS client and server. By default the NFS client establishes only one TCP connection, and all RPCs are multiplexed over this connection. In some cases this limits the bandwidth of sequential reads/writes. Increasing `nconnect` (maximum value 16) can solve this problem.

In particular, the `noatime` / `relatime` settings have no effect on NFS [3]; the NFS client always caches atime changes.

Some tutorials recommend modifying `rsize` and `wsize`. In NFSv4.2 these two values are already negotiated to their maximum value `1048576` by default, so there is no need to change them manually; you only need to check whether they were negotiated correctly.

According to [4], `sunrpc.tcp_max_slot_table_entries` may affect performance and can be increased appropriately (the default is `2`). In my testing, I found that when encountering a sustained small-file access workload on the order of tens of millions, NFS would sometimes hang. When I increased this parameter, the problem was resolved. Set `/etc/modprobe.d/sunrpc.conf`:

```ini
options sunrpc tcp_slot_table_entries=16384
```

Sometimes I encounter a problem where `nfsd` consumes a large amount of CPU and performance drops sharply, while a large number of `delegreturn` RPC calls are recorded. According to [5], this can be resolved by disabling `fs.leases-enable`. Set `/etc/sysctl.conf`:

```ini
fs.leases-enable = 0
```

When `nfsd` restarts for one reason or another, by default there is a 90s grace period for lock recovery, during which `nfsd` rejects all `open` requests, shown in the kernel log as:

```txt
[1073511.138061] NFSD: starting 90-second grace period (net f0000000)
```

In practice I found that this period can be reduced appropriately to lessen the impact of `nfsd` restarts. Set `/etc/default/nfs-kernel-server`:

```bash
# Options for rpc.svcgssd.
RPCSVCGSSDOPTS="--lease-time 10 --grace-time 10"
```

## Testing

TODO

## Conclusion

TODO

## References

[1] https://man.archlinux.org/man/exports.5.en#no_subtree_check

[2] https://man.archlinux.org/man/nfs.5.en#Using_NFS_over_UDP_on_high-speed_links

[3] https://man.archlinux.org/man/nfs.5.en#File_timestamp_maintenance

[4] https://learn.microsoft.com/en-us/azure/azure-netapp-files/performance-linux-concurrency-session-slots

[5] https://docs.gitlab.com/ee/administration/nfs.html#disable-nfs-server-delegation
