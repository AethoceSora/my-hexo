---
title: 如何在Nginx上支持HTTP/2
cover: https://s2.loli.net/2022/12/26/zN9BEclKUFpyDtw.png
tags: Devops
date: 2022/12/26 1:09:26
---

HTTP/2（原名HTTP 2.0）即超文本传输协议第二版，使用于万维网。HTTP/2主要基于SPDY协议，通过对HTTP头字段进行数据压缩、对数据传输采用多路复用和增加服务端推送等举措，来减少网络延迟，提高客户端的页面加载速度。HTTP/2没有改动HTTP的应用语义，仍然使用HTTP的请求方法、状态码和头字段等规则，它主要修改了HTTP的报文传输格式，通过引入二进制分帧实现性能的提升。

### 编译带有HTTP/2支持的Nginx

默认编译的 Nginx 并不包含 HTTP/2 模块，我们需要加入参数来编译.

我们首先记录下Nginx原先的编译配置：

```shell
/usr/local/nginx/sbin/nginx -V
```

得到下面的内容：

```shell
nginx version: nginx/1.20.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx --add-dynamic-module=../ngx-fancyindex-master/ --add-dynamic-module=../njs/nginx/ --with-http_addition_module --with-http_ssl_module --with-compat --with-http_stub_status_module
```



我们需要将到Nginx的`configure arguments`复制下来，到安装包的`./configure`中加入`--with-http_v2_module`，如果没有ssl支持，还需要加入`--with-http_ssl_module`

参考：

```shell
./configure --prefix=/usr/local/nginx --add-dynamic-module=../ngx-fancyindex-master/ --add-dynamic-module=../njs/nginx/ --with-http_addition_module --with-http_ssl_module --with-compat --with-http_stub_status_module --with-http_v2_module --with-http_sub_module
```

### Nginx配置文件

主要是配置 Nginx 的 `server` 块：在`listen 443 ssl`的后面加一个`http2`即可

```nginx
server {
listen 443 ssl http2 ;
server_name mirrors.qlu.edu.cn;

ssl_certificate /.../public.crt;
ssl_certificate_key /.../private.key;

...
```

