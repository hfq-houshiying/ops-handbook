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


## 在不删除客户端连接的情况下升级nginx

在升级过程中将使用以下信号：

`USR2`：生成一组新的 nginx `master/worker` 进程，而不会影响之前的设置。

`WINCH`：告诉 Nginx master 进程正常地停止其关联的 worker 进程。

`HUP`：    当 nginx 接收到 HUP 信号时，它会尝试先解析配置文件，如果成功，就应用新的配置文件(例如，重新打开日志文件或监听的套接字)。之后，nginx 运行新的工作进程并从容关闭旧的工作进程。通知工作进程关闭监听套接字，但是继续为当前连接的客户提供服务。所有的客户端的服务完成后，旧的工作进程被关闭。如果新的配置文件应用失败，nginx 将继续使用旧的配置文件进行工作。

`QUIT`：优雅的关闭 master/worker 进程。

`TERM`：快速关闭 master/worker 进程。

`KILL`：立即终止 master/worker 进程而不做任何清理。


### 查找 nginx 进程 pid

使用 ps aux 查找

```
$ ps aux|grep nginx
root     26378  0.0  0.4 125080  8672 ?        S    10:18   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data 26380  0.0  0.1 125456  3180 ?        S    10:18   0:00 nginx: worker process
```

查看 pid 文件查找

```
$cat /run/nginx.pid
```

如果有两个 Nginx 主进程正在运行，则旧进程将被移动到/run/nginx.pid.oldbin

### 启动新的 nginx

当 nginx 二进制文件更新替换完成之后, 发送 USR2 信号：

```
sudo kill -s USR2 `cat /run/nginx.pid`
```

会启动第二组 nginx  worker/master 进程

```
$ ps aux|grep nginx
root     26378  0.0  0.4 125080 10000 ?        S    10:18   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data 26380  0.0  0.1 125456  3180 ?        S    10:18   0:00 nginx: worker process
root     26420  0.0  0.4 125080  9688 ?        S    10:38   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data 26421  0.0  0.1 125456  3276 ?        S    10:38   0:00 nginx: worker process
vagrant  26423  0.0  0.0  12944   976 pts/0    S+   10:38   0:00 grep --color=auto nginx
```

原 pid 已经移动到 `/run/nginx.pid.oldbin` ,新的进程号在 `/run/nginx.pid`

```
$ tail -n +1 /run/nginx.pid*
==> /run/nginx.pid <==
26420

==> /run/nginx.pid.oldbin <==
26378
```

### 停止旧版本的 worker 进程

向旧版本的 nginx 发出 `WINCH` 信号，旧版本将处理完所有连接后停止接入新的连接

```
$ ps aux|grep nginx
root     26378  0.0  0.4 125080 10000 ?        S    10:18   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
root     26420  0.0  0.4 125080  9688 ?        S    10:38   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data 26421  0.0  0.1 125456  3276 ?        S    10:38   0:00 nginx: worker process
vagrant  26429  0.0  0.0  12944   968 pts/0    S+   10:43   0:00 grep --color=auto nginx
```

### 评估结果并采取后续步骤

如果升级成功，向旧的进程发送 `QUIT` 信号

```
sudo kill -s QUIT `cat /run/nginx.pid.oldbin`
```

旧的主进程将正常退出，只留下您的新 Nginx 的 master/worker 进程。此时，您已成功执行 Nginx 的版本升级，而不会中断客户端连接。


如果新版本遇到问题，回到旧版本, 向旧版本的 nginx 发送 `HUP` 信号来重新启动旧 nginx 的 worker 进程，此时这两组 nginx 都可以接受客户端的连接

```
sudo kill -s HUP `cat /run/nginx.pid.oldbin`
```
此时通过发送 `QUIT` 信号停止有缺陷的新版本,版本回退到旧版本。

```
sudo kill -s QUIT `cat /run/nginx.pid`
```
