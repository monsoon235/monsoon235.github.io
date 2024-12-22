---
title: "Using GPU accessible VS Code Server on UIUC Delta"
date: 2024-12-22
lang: en
tags:
  - vscode
  - network
  - linux
  - ssh
  - cloudflare
  - cloudflared
  - uiuc
  - delta
---

## Why writing this blog post

Many UIUC students rely on the [Delta](https://www.ncsa.illinois.edu/research/project-highlights/delta/) to access the GPU resources for their research. Delta provides 4 ssh-enabled login nodes, and lots of computing nodes with GPUs. Usually, we must ssh to the login node (by password and DUO 2FA OTP) first, and then use `srun` to request GPU resources to run our code. However, based on my experience, sometimes we could suffer many problems when using the Delta:

- **Unstable network connection**: Connection is lost frequently when the network is poor. Each time when the VS Code Remote lost connection, you must reenter the password and DUO 2FA OTP (you have to unlock your phone to get the OTP) to reconnect, which is annoying, time-consuming, and distracting.
- **Broken [OnDemand Code Server](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/ood/index.html)**: Although you can run VS COde Remote on the login nodes by ssh, there's no GPU for debugging, and the computing nodes are not accessible by ssh. The alternative ways include [OnDemand Jupyter Lab and Code Server](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/ood/index.html). But the functions of Jupiter Lab are limited, and the Code Server is broken -- When I try to request a Code Server on computing nodes, the system just queues and shows my request has been completed, **no running status**.

Due to the above problems, debugging GPU programs on Delta are struggling. That's why I wrote this blog post: by running private Code Server on computing nodes, and deploying a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) reverse proxy, you can say goodbye to these annoying problems.

## How to

My solution is based on an **observation** about the Delta: all login nodes and computing nodes are in a trusted network. There's no firewalls between them, which means you can access to any ports on the computing nodes from the login nodes.

The main steps of my solution are simple:

1. Use `srun` to get a tty on the computing node (e.g., on `gpua042` node).
2. Run a Code Server on the computing node. It will listen on `0.0.0.0:8080`.
3. Reverse proxy `gpua042:8080` to any port you have access. There are two approaches:
   - Use `ssh -L` to forward the port to your local machine.
   - Use Cloudflare Tunnel to reverse proxy the port to a public domain. This approach is more stable in poor network conditions.

### Run Code Server

Download the Code Server binary from the [Github repository](https://github.com/coder/code-server) (e.g., `code-server-4.96.2-linux-amd64.tar.gz`), and extract it. On the computing node, run:

```bash
cd code-server-4.96.2-linux-amd64/bin

## no auth
./code-server --bind-addr 0.0.0.0:8080 --auth none

## if port is exposed to untrusted network, use password auth
## password can be modified in ~/.config/code-server/config.yaml
./code-server --bind-addr 0.0.0.0:8080
```

### Access Code Server

#### SSH Port Forwarding

`ssh -L` can forward a local port to a remote port. Run:

```bash
ssh -L 127.0.0.1:8080:gpua042:8080 username@login.delta.ncsa.illinois.edu
```

Then open `http://127.0.0.1:8080` in your browser, and enjoy the Code Server!

#### Cloudflare Tunnel

Cloudflare Tunnel is more stable when your computer suffer from poor network connection. But it requires a domain name.

TODO
