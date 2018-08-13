# nginx-uwsgi-test

## virtualenv

创建项目目录，并创建一个独立的运行环境，命名为 `venv`,`--no-site-packages` 不会将系统已存在的 pip 包导入进来：

```
mkdir myproject
cd myproject
virtualenv --no-site-packages venv
```

使用 `source`进入该环境:

```
source venv/bin/activate
```

安装第三方包:

```
pip install django
```

创建 django demo 项目：

```
django-admin.py startproject demosite
cd demosite
python manage.py runserver 0.0.0.0:8002
```

测试 uwsgi

```
uwsgi --http 10.0.8.100:9090  --socket 127.0.0.1:9090 --file demosite/wsgi.py --pythonpath /home/vagrant/myproject/venv/lib/python2.7/site-packages
```

创建 uwsgi.ini

```
# uwsig使用配置文件启动
[uwsgi]
# 项目目录
chdir=/home/vagrant/myproject/demosite
# 指定sock的文件路径       
socket=127.0.0.1:9090
http=10.0.8.100:9090
wsgi-file=/home/vagrant/myproject/demosite/demosite/wsgi.py
# 进程个数       
workers=5
pidfile=/home/vagrant/myproject/demosite/uwsgi.pid
# 启动uwsgi的用户名和用户组
uid=vagrant
gid=vagrant
# 启用主进程
master=true
# 自动移除unix Socket和pid文件当服务停止的时候
vacuum=true
# 序列化接受的内容，如果可能的话
thunder-lock=true
# 启用线程
enable-threads=true
# 设置自中断时间
harakiri=30
# 设置缓冲
post-buffering=4096
# 设置日志目录
daemonize=/home/vagrant/myproject/demosite/uwsgi.log
# python packages
pythonpath = /home/vagrant/myproject/venv/lib/python2.7/site-packages
```

启动 uwsgi

```
uwsgi --ini uwsgi.ini
```

nginx 反代 uwsgi

```
server {
        listen       80;
        server_name  uwsgi-test;
        
        location / {            
            include  uwsgi_params;
            uwsgi_pass  127.0.0.1:9090;
            index  index.html index.htm;
            client_max_body_size 35m;
        }
    }
```

退出 virtualenv 环境

```
deactivate
```
