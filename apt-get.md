## 更新索引

更新本地软件包索引，列表在 `/etc/apt/sources.list` 中：

```
sudo apt-get update
```

## 安装软件包

```
sudo apt-get install package
```


## 升级软件包

当使用 `upgrade` 会从 `/etc/apt/sources.list` 中，检索最新版本，当使用 `upgrade` 升级时不会去新安装或删除别的依赖包，如果这个软件包的安装涉及到了需要安装新的软件包，或者删除旧的软件包，将不被允许使用 `upgrade` 更新。

```
sudo apt-get upgrade
```

dist-upgrade 除了执行升级功能外，还可以智能地处理新的依赖关系,升级时需要安装新的软件包，或者删除旧的软件包，将自动安装或删除，内核更新可以使用

```
sudo apt-get dist-upgrade
```

升级系统版本

```
do-release-upgrade
```

## 清理

删除软件包，不删除配置文件

```
sudo apt-get remove package
```

删除包及配置文件

```
sudo apt-get purge package
```

清理已下载的安装包，目录为 `/var/cache/apt/archives`

```
apt-get clean
```

清理过时的，社区不维护的包，并不会清理下载的安装包，目录为 `/var/cache/apt/archives`

```
apt-get autoclean
```

清理不需要的依赖软件

```
apt-get autoremove
```

## 常用选项

```
-qq 不输出信息，错误除外
--no-install-recommends 只安装必要的文件
-y  自动应答
-f  修复依赖关系
```
