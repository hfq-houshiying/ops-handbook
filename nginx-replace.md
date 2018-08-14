## nginx 替换

### 统一 nginx 版本

nginx 版本选择稳定版本 `1.14.0`，或者早期版本的最后一个版本 `1.12.2`。

`1.14.0` 相对比 `1.12.2` 而言，从 `1.11.13` 开始进行新功能的开发，发布了新的版本 `1.13.0`, 而 `1.12.0` 相对 `1.11.13` 而言，停止了新功能的开发，`1.12.1` 和 `1.12.2` ，做了安全性修复，和 bug 修复。

`1.14.0` 相比 `1.12.2` 新功能如下：

- 允许后端进行 SSL 协商

- 邮件代理功能多了 `rcvbuf` 和 `sndbuf` 参数

- 可以使用 `return` 和 `error_page` 指令进行 308 重定向

- `ssl_protocols` 指令支持 `TLS v1.3` 版本

- 当有进程向 nginx 发送 signal 信号时， nginx 日志记录发送 signal 信号的 PID

- 可以使用 `hostname` 主机名，作为 `set_real_ip_from` 的参数

- 添加 `add_trailer` 指令

- 添加流量镜像模块 `ngx_http_mirror_module`

- 添加变量 `$ssl_client_escaped_cert`

- 当使用 `proxy_bind`， `fastcgi_bind` ，`memcached_bind` ，`scgi_bind`和 `transparent` 参数时，nginx会自动保留工作进程中的 `CAP_NET_RAW` 功能 `uwsgi_bind` 指令

- HTTP2 服务器支持 `http2_push` 和 `http2_push_preload` 指令

- 添加新模块 `ngx_http_grpc_module`

- `ngx_stream_ssl_preread_module` 模块添加新变量 `$ssl_preread_alpn_protocols`

- logformat 功能增加 `escape = none` 参数

- nginx 使用 `clock_gettime`（`CLOCK_MONOTONIC`）（如果可用），以避免在系统时间更改时错误触发超时

- `include` SSI指令的 `set` 参数现在允许对变量写入任意响应; `subrequest_output_buffer_size` 指令定义最大响应大小

- listen 的 `proxy_protocol` 支持 V2 版本


如对新功能无需求，目前确定版本为 `1.12.2` 版本


## 安装步骤

安装选择源码安装方式，安装步骤如下:

```
apt-get install libpcre3-dev libgd-dev libgeoip-dev
curl -sSLO http://nginx.org/download/nginx-1.12.2.tar.gz
tar xf nginx-1.12.2.tar.gz
cd nginx
./configure --with-cc-opt='-g -O2 -fPIE -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2' --with-ld-opt='-Wl,-Bsymbolic-functions -fPIE -pie -Wl,-z,relro -Wl,-z,now' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-debug --with-pcre-jit --with-ipv6 --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_addition_module --with-http_dav_module --with-http_geoip_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module --with-http_v2_module --with-http_sub_module --with-http_xslt_module --with-stream --with-stream_ssl_module --with-mail --with-mail_ssl_module --with-threads
make

#将 objs 目录下编译出的 nginx 拿到主机 /opt 目录
# 将 原 /usr/sbin/nginx  mv 成 /usr/sbin/nginx-old
将新版的 nginx 放到 /usr/sbin/nginx

查看 nginx 是否一个机器起多个nginx，如果没有
/usr/sbin/nginx -t
pkill nginx
/usr/sbin/nginx

如果有 nginx -c xxx.conf
```
