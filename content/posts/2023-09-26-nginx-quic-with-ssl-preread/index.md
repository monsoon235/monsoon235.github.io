---
title: Enabling QUIC in Nginx While Keeping SNI Routing
date: 2023-09-26
slug: 2023-09-26-nginx-quic-with-ssl-preread
tags:
  - nginx
  - quic
  - http3
  - tls
  - sni-routing
---

## Problem

Since version 1.25.0, Nginx's support for QUIC [has been merged into mainline](https://nginx.org/en/docs/quic.html). Users who want to try it out can simply use the official `nginx` docker image, which is very convenient.

However, the nginx on my server uses [SNI routing](https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html), driven by the needs of a new generation of TLS-based proxy protocols such as [Shadow TLS](https://github.com/ihciah/shadow-tls) and [Xray Reality](https://github.com/XTLS/REALITY). These proxy protocols cannot have their TLS layer handled by nginx on their behalf (unlike earlier protocols that could use gRPC/WebSocket and the like as their data transport). But in order to achieve the best camouflage effect, using the `443/tcp` port is necessary (the whitelisted target sites used for camouflage generally only serve HTTPS on the `443/tcp` port). Therefore, multiplexing the `443/tcp` port is necessary.

To make SNI routing and QUIC coexist, you only need to add `listen 443 quic` to each server in the original SNI routing configuration. An example configuration is shown below.

## Configuration

```nginx
http {
    
    # ...

    server {
        server_name example.com;

        # 443/tcp is already occupied by nginx stream, so it cannot be listened on again
        # listen 443 ssl http2 reuseport so_keepalive=on;
        # listen [::]:443 ssl http2 reuseport so_keepalive=on;

        # Listen on the 443/udp port and enable QUIC
        # ref: https://nginx.org/en/docs/http/ngx_http_v3_module.html
        listen 443 quic reuseport;
        listen [::]:443 quic reuseport;

        # Listen on a unix domain socket to accept connections forwarded from stream; a local port can also be used
        # Accept proxy_protocol, otherwise the connection source address shown in the log will all be unix:
        listen unix:/dev/shm/nginx-example.sock ssl http2 proxy_protocol;
        set_real_ip_from unix:;  # Only override the source address for connections coming from the unix domain socket
        real_ip_header proxy_protocol;

        add_header Alt-Svc 'h3=":443"; ma=86400';  # used to advertise the availability of HTTP/3

        # ...
    }

    server {
        server_name foo.example.com;

        # Multiple domains can share 443/udp
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

    # Route based on TLS SNI
    map $ssl_preread_server_name $name {
        example.com             unix:/dev/shm/nginx-example.sock;
        foo.example.com         unix:/dev/shm/nginx-example-foo.sock;
        learn.microsoft.com     127.0.0.1:8443;  # used for shadow-tls/xray-reality, etc.
        default                 unix:/dev/shm/nginx-default.sock;
    }

    server {
        # Listen on 443/tcp and route based on SNI
        listen 443 reuseport so_keepalive=on;
        listen [::]:443 reuseport so_keepalive=on;
        proxy_pass $name;
        ssl_preread on;
        proxy_protocol on;
    }

}
```

## Testing

Currently, the mainline of `curl`/`wget` does not yet support QUIC. You can use the `ymuski/curl-http3` docker image:

```shell
$ docker run -it --rm ymuski/curl-http3 curl https://static.monsoon-cs.moe/public/ --http3 -IL

HTTP/3 200
server: nginx/1.25.2
date: Tue, 26 Sep 2023 14:52:29 GMT
content-type: text/html; charset=utf-8
strict-transport-security: max-age=63072000
alt-svc: h3=":443"; ma=86400
```

## References

- [https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html](https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html)
- [https://nginx.org/en/docs/http/ngx_http_v3_module.html](https://nginx.org/en/docs/http/ngx_http_v3_module.html)
