---
title: Building Proxy Service for Team
date: 2023-11-09
lang: en
tags:
  - proxy
  - warp
  - network
  - sing-box
  - clash
---

## Preface

Due to [Internet censorship in China](https://en.wikipedia.org/wiki/Internet_censorship_in_China) (known as *GFW*, *Great Firewall*, *防火长城*), many websites (e.g. Google, Google Scholar, Twitter) are blocked, and some websites (e.g. GitHub) suffer connectivity issues. In China, the means to circumvent (e.g. proxy) internet censorship is referred to as *翻墙* (means *climbing over the wall*).

In China, to freely access the Internet, a proxy is essential. Despite various commercial options available, they may not be suitable for everyone. Therefore, I have constructed a user-friendly and easy-to-maintain proxy system for my research group, as a part of my responsibilities as a system administrator.

## Target

1. **Easy to use**. Team members only need some simple configurations.The proxy client should be able to automatically update configuration.
2. **Stability**.
3. **Sufficient traffic**, to download large datasets.
4. **Low Latency**, to provide good experience for web.
5. **Low Cost**.
6. **Easy to maintain**. Frequent maintenance is unacceptable, and only simple changes of the configuration are required for new function.
7. **Concealment**. The cat-and-mouse game between GFW and anti-censorship tools has been escalating. Ten years ago (2013), only an OpenVPN client was all your need to ["Across the Great Wall and reach every corner in the world"](https://www.cnnic.com.cn/IDR/hlwfzdsj/201306/t20130628_40563.htm). Now, you must use much more sophisticated solutions to prevent your "unusual" traffic from being detected by GFW. According to [GFW Report](https://gfw.report/), popular [Shadowsocks](https://shadowsocks.org/) (a proxy protocol which simply encrypt all traffic using pre-shared key) was [detected and blocked](https://gfw.report/blog/gfw_shadowsocks/), and the TLS-based proxy also [encountered large-scale blocking in Oct 2022](https://github.com/net4people/bbs/issues/129). The tools and protocols used must be concealed enough to allow the service to run for a long time.
