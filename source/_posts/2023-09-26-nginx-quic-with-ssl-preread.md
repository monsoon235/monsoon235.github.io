---
title: Nginx 启用 QUIC 并和 SNI 分流共存
date: 2023-09-26
lang: zh-CN
tags: nginx,quic,http3,tls,sni-routing
---

## 问题

Nginx 自从 1.25.0 版本以来对 QUIC 的支持[已被合并入 mainline](https://nginx.org/en/docs/quic.html)，对于想体验的用户而言可以直接使用官方发布的 `nginx` docker 镜像，非常方便。

但是我的服务器上的 nginx 使用了 [SNI 分流](https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html)，源于 [Shadow TLS](https://github.com/ihciah/shadow-tls) 和 [Xray Reality](https://github.com/XTLS/REALITY) 等新一代基于 TLS 的代理协议的需求。这些代理协议并不能由 nginx 代为处理 TLS 层（和之前可以使用 gPRC/WebSocket 等作为数据传输方式的协议不同），但为了实现最好的伪装效果，使用 `443/tcp` 端口是有必要的（伪装的白名单目标网站一般情况下也只会在 `443/tcp` 端口开放 HTTPS 服务）。因此 `443/tcp` 端口的复用是必要的。

如果要让 SNI 分流和 QUIC 共存，在原来的 SNI 分流配置上只需要给每个 server 加上 `listen 443 quic` 即可。示例配置如下。

## 配置

```nginx
http {
    
    # ...

    server {
        server_name example.com;

        # 443/tcp 已经被 nginx stream 占用，不能再次监听
        # listen 443 ssl http2 reuseport so_keepalive=on;
        # listen [::]:443 ssl http2 reuseport so_keepalive=on;

        # 监听 443/udp 端口并启用 QUIC
        # ref: https://nginx.org/en/docs/http/ngx_http_v3_module.html
        listen 443 quic reuseport;
        listen [::]:443 quic reuseport;

        # 监听 unix domain socket 以接受 stream 传送过来的连接，也可以使用本地端口
        # 接受 proxy_protocol，否则 log 中显示的链接源地址都是 unix:
        listen unix:/dev/shm/nginx-example.sock ssl http2 proxy_protocol;
        set_real_ip_from unix:;  # 只对于来自 unix domain socket 的连接覆盖其源地址
        real_ip_header proxy_protocol;

        add_header Alt-Svc 'h3=":443"; ma=86400';  # used to advertise the availability of HTTP/3

        # ...
    }

    server {
        server_name foo.example.com;

        # 可以多个域名共享 443/udp
        listen 443 quic;
        listen [::]:443 quic;

        listen unix:/dev/shm/nginx-example-foo.sock ssl http2 proxy_protocol;
        set_real_ip_from unix:;
        real_ip_header proxy_protocol;

        add_header Alt-Svc 'h3=":443"; ma=86400';  # used to advertise the availability of HTTP/3

        # ...
    }
}

stream {

    # ...

    # 根据 TLS SNI 分流
    map $ssl_preread_server_name $name {
        example.com             unix:/dev/shm/nginx-example.sock;
        foo.example.com         unix:/dev/shm/nginx-example-foo.sock;
        learn.microsoft.com     127.0.0.1:8443;  # 用于 shadow-tls/xray-reality 等
        default                 unix:/dev/shm/nginx-default.sock;
    }

    server {
        # 监听 443/tcp 并根据 SNI 分流
        listen 443 reuseport so_keepalive=on;
        listen [::]:443 reuseport so_keepalive=on;
        proxy_pass $name;
        ssl_preread on;
        proxy_protocol on;
    }

}
```

## 测试

目前 `curl`/`wget` mainline 还没有支持 QUIC，可以使用 `ymuski/curl-http3` 这个 docker 镜像：

```shell
$ docker run -it --rm ymuski/curl-http3 curl https://static.monsoon-cs.moe/public/ --http3 -IL

HTTP/3 200
server: nginx/1.25.2
date: Tue, 26 Sep 2023 14:52:29 GMT
content-type: text/html; charset=utf-8
strict-transport-security: max-age=63072000
alt-svc: h3=":443"; ma=86400
```

## 参考

- [https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html](https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html)
- [https://nginx.org/en/docs/http/ngx_http_v3_module.html](https://nginx.org/en/docs/http/ngx_http_v3_module.html)
