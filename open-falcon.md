## 环境初始化

安装 golang

```
./install-go.sh --install
```

安装 redis mysql

```
sudo apt-get install  -y redis-server mariadb-server
```

初始化表结构

```
cd /tmp/ && git clone https://github.com/open-falcon/falcon-plus.git 
cd /tmp/falcon-plus/scripts/mysql/db_schema/
mysql -h 127.0.0.1 -u root -p123456 < 1_uic-db-schema.sql
mysql -h 127.0.0.1 -u root -p123456 < 2_portal-db-schema.sql
mysql -h 127.0.0.1 -u root -p123456 < 3_dashboard-db-schema.sql
mysql -h 127.0.0.1 -u root -p123456 < 4_graph-db-schema.sql
mysql -h 127.0.0.1 -u root -p123456 < 5_alarms-db-schema.sql
rm -rf /tmp/falcon-plus/
```

Mysql 更改绑定地址

```
sudo sed -i  's@127.0.0.1@0.0.0.0@g' /etc/mysql/my.cnf
sudo service mysql restart
mysql -u root -p123456 -e "update mysql.user set host='%' where user = 'root' and host = '127.0.0.1';"
mysql -u root -p123456 -e "flush privileges;"
```

redis 修改绑定地址

```
sudo sed  -i 's@127.0.0.1@0.0.0.0@g' /etc/redis/redis.conf
sudo service redis-server restart
```

编译 open-falcon
```
go get github.com/open-falcon/falcon-plus
cd $GOPATH/src/github.com/open-falcon/falcon-plus/
make all
make pack
```

安装 open-falcon
```
sudo mkdir /usr/local/open-falcon
sudo tar xf open-falcon-v0.2.1.tar.gz -C /usr/local/open-falcon/
```

##  安装 agent

安装 agent, 配置文件必须叫cfg.json, agent用于采集机器负载监控指标，比如cpu.idle、load.1min、disk.io.util等等，每隔60秒push给Transfer。agent提供了一个http接口/v1/push用于接收用户手工push的一些数据，然后通过长连接迅速转发给Transfer。

```
vim  /usr/local/open-falcon/agent/config/cfg.json
{
    "debug": true,  # 控制一些debug信息的输出，生产环境通常设置为false
    "hostname": "", # agent采集了数据发给transfer，endpoint就设置为了hostname，默认通过`hostname`获取，如果配置中配置了hostname，就用配置中的
    "ip": "", # agent与hbs心跳的时候会把自己的ip地址发给hbs，agent会自动探测本机ip，如果不想让agent自动探测，可以手工修改该配置
    "plugin": {
        "enabled": false, # 默认不开启插件机制
        "dir": "./plugin",  # 把放置插件脚本的git repo clone到这个目录
        "git": "https://github.com/open-falcon/plugin.git", # 放置插件脚本的git repo地址
        "logs": "./logs" # 插件执行的log，如果插件执行有问题，可以去这个目录看log
    },
    "heartbeat": {
        "enabled": true,  # 此处enabled要设置为true
        "addr": "0.0.0.0:6030", # hbs的地址，端口是hbs的rpc端口
        "interval": 60, # 心跳周期，单位是秒
        "timeout": 1000 # 连接hbs的超时时间，单位是毫秒
    },
    "transfer": {
        "enabled": true, 
        "addrs": [
            "0.0.0.0:8433"
        ],  # transfer的地址，端口是transfer的rpc端口, 可以支持写多个transfer的地址，agent会保证HA
        "interval": 60, # 采集周期，单位是秒，即agent一分钟采集一次数据发给transfer
        "timeout": 1000 # 连接transfer的超时时间，单位是毫秒
    },
    "http": {
        "enabled": true,  # 是否要监听http端口
        "listen": ":1988",
        "backdoor": false
    },
    "collector": {
        "ifacePrefix": ["eth", "em"], # 默认配置只会采集网卡名称前缀是eth、em的网卡流量，配置为空就会采集所有的，lo的也会采集。可以从/proc/net/dev看到各个网卡的流量信息
        "mountPoint": []
    },
    "default_tags": {
    },
    "ignore": {  # 默认采集了200多个metric，可以通过ignore设置为不采集
        "cpu.busy": true,
        "df.bytes.free": true,
        "df.bytes.total": true,
        "df.bytes.used": true,
        "df.bytes.used.percent": true,
        "df.inodes.total": true,
        "df.inodes.free": true,
        "df.inodes.used": true,
        "df.inodes.used.percent": true,
        "mem.memtotal": true,
        "mem.memused": true,
        "mem.memused.percent": true,
        "mem.memfree": true,
        "mem.swaptotal": true,
        "mem.swapused": true,
        "mem.swapfree": true
    }
}
```

启动 
```
cd /usr/local/open-falcon 
./open-falcon start agent  启动进程
./open-falcon stop agent  停止进程
./open-falcon monitor agent  查看日志

./agent/bin/falcon-agent --check  检查 agent 状态,也可以通过 1988 端口检查
```

## 安装 transfer

transfer是数据转发服务。它接收agent上报的数据，然后按照哈希规则进行数据分片、并将分片后的数据分别push给graph&judge等组件。

```
vim  transfer/config/cfg.json 
{
    "debug": true, #如果为true，日志中会打印debug信息
    "minStep": 30, #允许上报的数据最小间隔，默认为30秒
    "http": {
        "enabled": true, #表示是否开启该http端口，该端口为控制端口，主要用来对transfer发送控制命令、统计命令、debug命令等
        "listen": "0.0.0.0:6060" #表示监听的http端口
    },
    "rpc": {
        "enabled": true, #表示是否开启该jsonrpc数据接收端口, Agent发送数据使用的就是该端口
        "listen": "0.0.0.0:8433" #表示监听的http端口
    },
    "socket": { #即将被废弃,请避免使用
        "enabled": true, 
        "listen": "0.0.0.0:4444",
        "timeout": 3600
    },
    "judge": {
        "enabled": true, #表示是否开启向judge发送数据
        "batch": 200,  #数据转发的批量大小，可以加快发送速度，建议保持默认值
        "connTimeout": 1000, #单位是毫秒，与后端建立连接的超时时间，可以根据网络质量微调，建议保持默认
        "callTimeout": 5000, #单位是毫秒，发送数据给后端的超时时间，可以根据网络质量微调，建议保持默认
        "maxConns": 32, #连接池相关配置，最大连接数，建议保持默
        "maxIdle": 32,  #连接池相关配置，最大空闲连接数，建议保持默认
        "replicas": 500, #这是一致性hash算法需要的节点副本数量，建议不要变更，保持默认即可
        "cluster": {  #key-value形式的字典，表示后端的judge列表，其中key代表后端judge名字，value代表的是具体的ip:port
            "judge-00" : "0.0.0.0:6080" 
        }
    },
    "graph": {
        "enabled": true, #表示是否开启向graph发送数据
        "batch": 200, #数据转发的批量大小，可以加快发送速度，建议保持默认值
        "connTimeout": 1000, #单位是毫秒，与后端建立连接的超时时间，可以根据网络质量微调，建议保持默认
        "callTimeout": 5000, #单位是毫秒，发送数据给后端的超时时间，可以根据网络质量微调，建议保持默认
        "maxConns": 32, #连接池相关配置，最大连接数，建议保持默认
        "maxIdle": 32,  #连接池相关配置，最大空闲连接数，建议保持默认
        "replicas": 500, #这是一致性hash算法需要的节点副本数量，建议不要变更，保持默认即可
        "cluster": { #key-value形式的字典，表示后端的graph列表，其中key代表后端graph名字，value代表的是具体的ip:port(多个地址用逗号隔开, transfer会将同一份数据发送至各个地址，利用这个特性可以实现数据的多重备份)
            "graph-00" : "0.0.0.0:6070"
        }
    },
    "tsdb": {
        "enabled": false, #表示是否开启向open tsdb发送数据
        "batch": 200, #数据转发的批量大小，可以加快发送速度
        "connTimeout": 1000, #单位是毫秒，与后端建立连接的超时时间，可以根据网络质量微调，建议保持默认
        "callTimeout": 5000, #单位是毫秒，发送数据给后端的超时时间，可以根据网络质量微调，建议保持默认
        "maxConns": 32, #连接池相关配置，最大连接数，建议保持默认
        "maxIdle": 32,  #连接池相关配置，最大空闲连接数，建议保持默认
        "retry": 3,     #连接后端的重试次数和发送数据的重试次数
        "address": "127.0.0.1:8088" #tsdb地址或者tsdb集群vip地址, 通过tcp连接tsdb.
    }
}

# 启动服务
./open-falcon start transfer

# 校验服务,这里假定服务开启了6060的http监听端口。检验结果为ok表明服务正常启动。
curl -s "127.0.0.1:6060/health"

# 停止服务
./open-falcon stop transfer

# 查看日志
./open-falcon monitor transfer
```

部署完成transfer组件后，请修改agent的配置，使其指向正确的transfer地址。在安装完graph和judge后，请修改transfer的相应配置、使其能够正确寻址到这两个组件。

## 安装 graph

graph是存储绘图数据的组件。graph组件 接收transfer组件推送上来的监控数据，同时处理api组件的查询请求、返回绘图数据。

```
vim graph/config/cfg.json
{
    "debug": false, //true or false, 是否开启debug日志
    "http": {
        "enabled": true, //true or false, 表示是否开启该http端口，该端口为控制端口，主要用来对graph发送控制命令、统计命令、debug命令
        "listen": "0.0.0.0:6071" //表示监听的http端口
    },
    "rpc": {
        "enabled": true, //true or false, 表示是否开启该rpc端口，该端口为数据接收端口
        "listen": "0.0.0.0:6070" //表示监听的rpc端口
    },
    "rrd": {
        "storage": "./data/6070" // 历史数据的文件存储路径（如有必要，请修改为合适的路）
    },
    "db": {
        "dsn": "root:@tcp(127.0.0.1:3306)/graph?loc=Local&parseTime=true", //MySQL的连接信息，默认用户名是root，密码为空，host为127.0.0.1，database为graph（如有必要，请修改)
        "maxIdle": 4  //MySQL连接池配置，连接池允许的最大连接数，保持默认即可
    },
    "callTimeout": 5000,  //RPC调用超时时间，单位ms
    "migrate": {  //扩容graph时历史数据自动迁移
        "enabled": false,  //true or false, 表示graph是否处于数据迁移状态
        "concurrency": 2, //数据迁移时的并发连接数，建议保持默认
        "replicas": 500, //这是一致性hash算法需要的节点副本数量，建议不要变更，保持默认即可（必须和transfer的配置中保持一致）
        "cluster": { //未扩容前老的graph实例列表
            "graph-00" : "127.0.0.1:6070"
        }
    }
}
```

部署完graph组件后，请修改transfer和api的配置，使这两个组件可以寻址到graph

```
# 启动服务
./open-falcon start graph

# 停止服务
./open-falcon stop graph

# 查看日志
./open-falcon monitor graph
```

##安装 api

```
vim api/config/cfg.json
{
    "log_level": "debug",
    "db": {  //数据库相关的连接配置信息
        "faclon_portal": "root:@tcp(127.0.0.1:3306)/falcon_portal?charset=utf8&parseTime=True&loc=Local",
        "graph": "root:@tcp(127.0.0.1:3306)/graph?charset=utf8&parseTime=True&loc=Local",
        "uic": "root:@tcp(127.0.0.1:3306)/uic?charset=utf8&parseTime=True&loc=Local",
        "dashboard": "root:@tcp(127.0.0.1:3306)/dashboard?charset=utf8&parseTime=True&loc=Local",
        "alarms": "root:@tcp(127.0.0.1:3306)/alarms?charset=utf8&parseTime=True&loc=Local",
        "db_bug": true
    },
    "graphs": {  // graph模块的部署列表信息
        "cluster": {
            "graph-00": "127.0.0.1:6070"
        },
        "max_conns": 100,
        "max_idle": 100,
        "conn_timeout": 1000,
        "call_timeout": 5000,
        "numberOfReplicas": 500
    },
    "metric_list_file": "./api/data/metric",
    "web_port": ":8080",  // http监听端口
    "access_control": true, // 如果设置为false，那么任何用户都可以具备管理员权限
    "salt": "pleaseinputwhichyouareusingnow",  //数据库加密密码的时候的salt
    "skip_auth": false, //如果设置为true，那么访问api就不需要经过认证
    "default_token": "default-token-used-in-server-side",  //用于服务端各模块间的访问授权
    "gen_doc": false,
    "gen_doc_path": "doc/module.html"
}
```

启动

```
# 启动服务
./open-falcon start api

# 停止服务
./open-falcon stop api

# 查看日志
./open-falcon monitor api
```

部署完成api组件后，请修改dashboard组件的配置、使其能够正确寻址到api组件。
请确保api组件的graph列表 与 transfer的配置 一致。

## 安装 dashboard

初始化环境
```
export HOME=/home/work
export WORKSPACE=$HOME/open-falcon
sudo mkdir -p $WORKSPACE
sudo apt-get install libldap2-dev libsasl2-dev

cd $WORKSPACE
sudo git clone https://github.com/open-falcon/dashboard.git
sudo chown -R vagrant.vagrant dashboard

cd dashboard
virtualenv ./env
./env/bin/pip install -r pip_requirements.txt -i http://mirrors.aliyun.com/pypi/simple/
```

修改配置
```
dashboard的配置文件为： 'rrd/config.py'，请根据实际情况修改


sudo vim rrd/config.py

## API_ADDR 表示后端api组件的地址
API_ADDR = "http://127.0.0.1:8080/api/v1" 

## 根据实际情况，修改PORTAL_DB_*, 默认用户名为root，默认密码为""
## 根据实际情况，修改ALARM_DB_*, 默认用户名为root，默认密码为""
```

启动

```
#开发环境
./env/bin/python wsgi.py

#生产环境
bash control start

#查看日志
bash control tail
```

用户管理

```
dashbord没有默认创建任何账号包括管理账号，需要你通过页面进行注册账号。
想拥有管理全局的超级管理员账号，需要手动注册用户名为root的账号（第一个帐号名称为root的用户会被自动设置为超级管理员）。
超级管理员可以给普通用户分配权限管理。

小提示：注册账号能够被任何打开dashboard页面的人注册，所以当给相关的人注册完账号后，需要去关闭注册账号功能。只需要去修改api组件的配置文件cfg.json，将signup_disable配置项修改为true，重启api即可。当需要给人开账号的时候，再将配置选项改回去，用完再关掉即可。
```

## 安装 HBS(Heartbeat Server)

心跳服务器，公司所有agent都会连到HBS，每分钟发一次心跳请求。

```
vim hbs/config/cfg.json

{
    "debug": true,
    "database": "root:password@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true", # Portal的数据库地址
    "hosts": "", # portal数据库中有个host表，如果表中数据是从其他系统同步过来的，此处配置为sync，否则就维持默认，留空即可
    "maxIdle": 100,
    "listen": ":6030", # hbs监听的rpc地址
    "trustable": [""],
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6031" # hbs监听的http地址
    }
}
```

启动

```
# 启动
./open-falcon start hbs

# 停止
./open-falcon stop hbs

# 查看日志
./open-falcon monitor hbs
```

如果你先部署了agent，后部署的hbs，那咱们部署完hbs之后需要回去修改agent的配置，把agent配置中的heartbeat部分enabled设置为true，addr设置为hbs的rpc地址。如果hbs的配置文件维持默认，rpc端口就是6030，http端口是6031，agent中应该配置为hbs的rpc端口，小心别弄错了。


## Judge

Judge用于告警判断，agent将数据push给Transfer，Transfer不但会转发给Graph组件来绘图，还会转发给Judge用于判断是否触发告警。

配置

```
vim judge/config/cfg.json
{
    "debug": true,
    "debugHost": "nil",
    "remain": 11,
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6081"
    },
    "rpc": {
        "enabled": true,
        "listen": "0.0.0.0:6080"
    },
    "hbs": {
        "servers": ["127.0.0.1:6030"], # hbs最好放到lvs vip后面，所以此处最好配置为vip:port
        "timeout": 300,
        "interval": 60
    },
    "alarm": {
        "enabled": true,
        "minInterval": 300, # 连续两个报警之间至少相隔的秒数，维持默认即可
        "queuePattern": "event:p%v",
        "redis": {
            "dsn": "127.0.0.1:6379", # 与alarm、sender使用一个redis
            "maxIdle": 5,
            "connTimeout": 5000,
            "readTimeout": 5000,
            "writeTimeout": 5000
        }
    }
}
```

启动

```
# 启动
./open-falcon start judge

# 停止
./open-falcon stop judge

# 查看日志
./open-falcon monitor judge
```

## Alarm

alarm模块是处理报警event的，judge产生的报警event写入redis，alarm从redis读取处理，并进行不同渠道的发送。

```
vim alarm/config/cfg.json

{
    "log_level": "debug",
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:9912"
    },
    "redis": {
        "addr": "127.0.0.1:6379",
        "maxIdle": 5,
        "highQueues": [
            "event:p0",
            "event:p1",
            "event:p2"
        ],
        "lowQueues": [
            "event:p3",
            "event:p4",
            "event:p5",
            "event:p6"
        ],
        "userIMQueue": "/queue/user/im",
        "userSmsQueue": "/queue/user/sms",
        "userMailQueue": "/queue/user/mail"
    },
    "api": {
        "im": "http://127.0.0.1:10086/wechat",  //微信发送网关地址
        "sms": "http://127.0.0.1:10086/sms",  //短信发送网关地址
        "mail": "http://127.0.0.1:10086/mail", //邮件发送网关地址
        "dashboard": "http://127.0.0.1:8081",  //dashboard模块的运行地址
        "plus_api":"http://127.0.0.1:8080",   //falcon-plus api模块的运行地址
        "plus_api_token": "default-token-used-in-server-side" //用于和falcon-plus api模块服务端之间的通信认证token
    },
    "falcon_portal": {
        "addr": "root:@tcp(127.0.0.1:3306)/alarms?charset=utf8&loc=Asia%2FChongqing",
        "idle": 10,
        "max": 100
    },
    "worker": {
        "im": 10,
        "sms": 10,
        "mail": 50
    },
    "housekeeper": {
        "event_retention_days": 7,  //报警历史信息的保留天数
        "event_delete_batch": 100
    }
}
```

进程管理
```
# 启动
./open-falcon start alarm

# 停止
./open-falcon stop alarm

# 查看日志
./open-falcon monitor alarm
```

## Nodata

nodata用于检测监控数据的上报异常。nodata和实时报警judge模块协同工作，过程为: 配置了nodata的采集项超时未上报数据，nodata生成一条默认的模拟数据；用户配置相应的报警策略，收到mock数据就产生报警。采集项上报异常检测，作为judge模块的一个必要补充，能够使judge的实时报警功能更加可靠、完善。

```
# 修改配置, 配置项含义见下文
vim nodata/config/cfg.json

{
    "debug": true,
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6090"
    },
    "plus_api":{
        "connectTimeout": 500,
        "requestTimeout": 2000,
        "addr": "http://127.0.0.1:8080",  #falcon-plus api模块的运行地址
        "token": "default-token-used-in-server-side"  #用于和falcon-plus api模块的交互认证token
    },
    "config": {
        "enabled": true,
        "dsn": "root:@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true&wait_timeout=604800",
        "maxIdle": 4
    },
    "collector":{
        "enabled": true,
        "batch": 200,
        "concurrent": 10
    },
    "sender":{
        "enabled": true,
        "connectTimeout": 500,
        "requestTimeout": 2000,
        "transferAddr": "127.0.0.1:6060",  #transfer的http监听地址,一般形如"domain.transfer.service:6060"
        "batch": 500
    }
}


# 启动服务
./open-falcon start nodata

# 停止服务
./open-falcon stop nodata

# 检查日志
./open-falcon monitor nodata
```

## Aggregator

集群聚合模块。聚合某集群下的所有机器的某个指标的值，提供一种集群视角的监控体验。

```
# 修改配置, 配置项含义见下文
vim aggregator/config/cfg.json
{
    "debug": true,
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6055"
    },
    "database": {
        "addr": "root:@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true",
        "idle": 10,
        "ids": [1, -1],
        "interval": 55
    },
    "api": {
        "connect_timeout": 500,
        "request_timeout": 2000,
        "plus_api": "http://127.0.0.1:8080",  #falcon-plus api模块的运行地址
        "plus_api_token": "default-token-used-in-server-side", #和falcon-plus api 模块交互的认证token
        "push_api": "http://127.0.0.1:1988/v1/push"  #push数据的http接口，这是agent提供的接口
    }
}

```

启动

```
# 启动服务
./open-falcon start aggregator

# 检查log
./open-falcon monitor aggregator

# 停止服务
./open-falcon stop aggregator
```

## 安装邮件网关

```
wget http://cactifans.hi-www.com/open-falcon/mail-provider.tar.gz
mkdir -p mail-provider
tar zxvf mail-provider.tar.gz  -C mail-provider
cd mail-provider

vim mail-provider/cfg.json

{
    "debug": true,
    "http": {
        "listen": "0.0.0.0:4000",
        "token": ""
    },
    "smtp": {
        "addr": "mail.example.com:25",
        "username": "falcon@example.com",
        "password": "123456",
        "from": "falcon@example.com"
    }
}



bash control start
```
安装完成之后修改 alarm 的邮件地址



