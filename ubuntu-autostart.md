# ubuntu auto-start

在 centos 中使用 chkconfig 去管理，而在 ubuntu 中使用 `update-rc.d`。

列出当前正在运行的进程

```
service --status-all
```

如果不属于 `is_ignored_file`，并且有执行权限，则检查：

如果没有 status 选项，则显示 `[?]`：

```
if ! is_ignored_file "${SERVICE}" \
                    && [ -x "${SERVICEDIR}/${SERVICE}" ]; then
                        if ! grep -qs "\(^\|\W\)status)" "$SERVICE"; then
                          #printf " %s %-60s %s\n" "[?]" "$SERVICE:" "unknown" 1>&2
                          echo " [ ? ]  $SERVICE" 1>&2
                          continue
```

如果有 `status` 选项，返回值是 `0` ,并且有输出，则显示 `[+]`

```
else
                          out=$(env -i LANG="$LANG" PATH="$PATH" TERM="$TERM" "$SERVICEDIR/$SERVICE" status 2>&1)
                          if [ "$?" = "0" -a -n "$out" ]; then
                            #printf " %s %-60s %s\n" "[+]" "$SERVICE:" "running"
                            echo " [ + ]  $SERVICE"
                            continue
```

如果有 `status` 选项，返回值是不为 0 ，或者没有输出，则显示 `[-]`

```
 else
                            #printf " %s %-60s %s\n" "[-]" "$SERVICE:" "NOT running"
                            echo " [ - ]  $SERVICE"
                            continue
                          fi
```

设置开机启动, defaults 会在 `# Default-Start:     2 3 4 5` 启动级别，开机启动 nginx

```
vagrant@ubuntu:~$ sudo update-rc.d  nginx defaults
 Adding system startup for /etc/init.d/nginx ...
   /etc/rc0.d/K20nginx -> ../init.d/nginx
   /etc/rc1.d/K20nginx -> ../init.d/nginx
   /etc/rc6.d/K20nginx -> ../init.d/nginx
   /etc/rc2.d/S20nginx -> ../init.d/nginx
   /etc/rc3.d/S20nginx -> ../init.d/nginx
   /etc/rc4.d/S20nginx -> ../init.d/nginx
   /etc/rc5.d/S20nginx -> ../init.d/nginx

```

删除开机启动

```
vagrant@ubuntu:~$ sudo update-rc.d -f  nginx remove
 Removing any system startup links for /etc/init.d/nginx ...
   /etc/rc0.d/K20nginx
   /etc/rc1.d/K20nginx
   /etc/rc2.d/S20nginx
   /etc/rc3.d/S20nginx
   /etc/rc4.d/S20nginx
   /etc/rc5.d/S20nginx
   /etc/rc6.d/K20nginx
```


写一个脚本测试开机启动，权限设置为 755

```
root@ubuntu:~# cat /etc/init.d/test 
#!/bin/bash


test_func() {
  while true;do
    sleep 10 && echo "$(date +%T)" >> /tmp/test.log
  done
}

case $1 in
    start)
        if ! [ -f /tmp/test.pid ];then
          test_func &
          echo "$!"  > /tmp/test.pid
        fi
        echo "start" ;;
    stop)
        echo "stop" 
        kill $(cat /tmp/test.pid)
        rm /tmp/test.pid ;;
    status)
        if [ -f /tmp/test.pid ];then
        echo "running"
        fi ;;
esac
```

测试开机启动

```
sudo update-rc.d  test defaults
sudo update-rc.d  -f test remove
```
