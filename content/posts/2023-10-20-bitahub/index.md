---
title: "Using an SSH Reverse Tunnel to Log Into BitaHub Containers and Hold GPUs Long-Term"
date: 2023-10-20
slug: 2023-10-20-bitahub
tags:
  - docker
  - ssh
---

## Problem

Every year before CVPR, GPUs are always in short supply, and we need to borrow cards from elsewhere. USTC provides [BitaHub](https://bitahub.ustc.edu.cn/) for on-campus users, but it suffers from the same shortage of cards before CVPR. At the same time, its job-submission-based usage model is very inconvenient: submitting jobs that occupy multiple cards often requires a long wait in the queue, and its data management approach is downright user-hostile.

As the server administrator for my group, in order to make my life easier before CVPR and to avoid repeating the 2021 pre-CVPR ordeal of scrambling to allocate resources, I needed to improve the BitaHub experience:

1. How to hold GPUs long-term to avoid repeatedly queuing (slightly unethical, but a measure born of necessity);
2. How to conveniently read data from our own servers, instead of being forced to use BitaHub's user-hostile data management model;
3. How to make the BitaHub GPU experience as close as possible to that of our group's servers, lowering migration costs and improving the flexibility of resource scheduling.

## Idea

Jobs in BitaHub run as docker containers, which gives us the possibility of configuring the environment we want inside the container, as long as we can somehow ssh into it.

After some investigation, I found that as long as the startup command does not stop running, a BitaHub container will keep running indefinitely and will not release its GPU resources. **At the same time, BitaHub containers have network access**, and the BitaHub web page even thoughtfully provides the ssh private key for the root user inside each job's container.

These facts give us an opportunity to exploit. All we need to do is run a tunnel program inside the container so that external parties can access port 22 of the container, and then we can log in and hold the resources long-term. Moreover, since the container has network access, we can also directly mount the file systems of other on-campus servers.

## Solution

The tunnel program I ended up choosing is `ssh`, which can create a reverse tunnel:

```shell
ssh -i <key_file> -F none -o "StrictHostKeyChecking no" -o "ServerAliveInterval 15" -v -N -R <port>:localhost:22 jump@<jumpserver>
```

On the `jumpserver`, configure a user `jump` and allow login with a specific private key, then somehow get the private key into the container (you could bake it directly into the image, but I chose a more convenient approach: create a BitaHub dataset to store it, and just add this dataset to every job).

The container's startup command is exactly the command above (considering network fluctuations, you can wrap it in a `while true` loop or use `autossh` to reconnect automatically). Once started, it creates a reverse tunnel on `<port>` of `<jumpserver>`, with `<port>` mapped to port `22` inside the container.

You can set `GatewayPorts yes` in the `sshd_config` of `<jumpserver>` so that the reverse tunnel listens on `0.0.0.0` instead of `127.0.0.1`. Otherwise, I would have to create a user on `<jumpserver>` for every person, or forward each port with `iptables`, which is far too tedious. Binding to `0.0.0.0` lets us access it directly from the existing VPN network.

There are many options for mounting a file system. Considering both security and convenience, I chose SSHFS. Exposing NFS directly to the public internet is too dangerous, while configuring NFS user authentication is too tedious. At the same time, the kernel that BitaHub uses to run containers neither loads the `wireguard` kmod nor maps `/dev/net/tun`, so we cannot use a VPN to protect data security. SSHFS can directly reuse the existing user authentication mechanism, and SSH traffic itself is also more likely to be let through by any potential data-center firewall.

Use the following command to mount SSHFS:

```shell
sshfs -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=30,ssh_command='ssh -p <dataserver_port> -i <key_file>' <user>@<dataserver>:/path /path
```

## Postscript

TODO
