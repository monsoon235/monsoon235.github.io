---
title: Catching Mining Virus
date: 2023-11-01
lang: en
tags:
  - linux
  - iptables
  - anti-virus
---

## Problem

On October 30, 2023, I received a warning message from the data center administrator, informing me that the firewall detected mining traffic sending from the server managed by me.

![](firewall_warning.png)

The "mining traffic" was a `bitcoin.sipa.be` DNS request sent to `223.5.5.5`.

Initially, I thought it was a simple task to find the virus process, just like my previous encounter with another mining virus. In that case, the hacker logged in the server by hacking a weak SSH password, gained root permission possibly by an privilege escalation vulnerability exploitation (it was a server running EOL Ubuntu 16.04). Then a cron job was set up to run a mining virus.

However, this time the situation was different. I couldn't find any suspicious processes, and there was no unusual GPU usage. Since I didn't deploy any monitoring programs to record historical processes and sockets, the investigation couldn't get started.

On October 31, I received the same warning again. Each time when mining traffic is detected, the firewall will block the server's outbound network. Loss of Internet will cause lots of troubles.

I suspected that someone may have suffered a **supply chain attack**, such as, downloading a Python package containing a virus, or cloning code from GitHub and running it without any check.

The immediate task is to identify who and which process was responsible.

## Solution

While I can't directly determine who or which process, I can block and log suspicious traffic for further investigation.

This job can be done by `iptables`:

```shell
# iptables -N LOGDROP                   # create a new chain
# iptables -A LOGDROP -j LOG --log-uid  # log info
# iptables -A LOGDROP -j DROP           # drop packet

# iptables -I OUTPUT 1 -p udp -m string --string "bitcoin" --algo bm -j LOGDROP     # match string "bitcoin" in udp packet
```

The `--log-uid` option can enable UID recording in `/var/log/kern.log`, for example:

```log
IN= OUT=wg0 SRC=10.1.92.3 DST=10.1.2.13 LEN=42 TOS=0x00 PREC=0x00 TTL=64 ID=23294 DF PROTO=UDP SPT=52328 DPT=2333 LEN=22 UID=2109 GID=2109
```

## Result

I'm waiting the next requests sent by virus.
