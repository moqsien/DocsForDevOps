# centos7使用lsyncd实时同步

1. lsyncd是lua语言封装了inotify和rsync的工具，采用了linux内核（2.6.13及以后）里的inotify处罚机制，然后通过rsync去差异同步，达到实时的效果。
2. 他完美解决了inotify+rsync海量文件同步带来的文件频繁发送文件列表的问题--通过时间延迟或累计触发事件次数实现。
3. lsyncd工作模式分为本地目录cp，本地目录rsync，远程目录rsync，远程目录rsyncssh。
4. github[地址](https://github.com/axkibe/lsyncd)，详细[说明](https://axkibe.github.io/lsyncd/)

## 安装

### 直接安装

```shell
yum install lsyncd
```

### 源码编译安装
1. 下载源码，并编译安装

```shell
# 未测试
uzip lsyncd-master.zip
cd lsyncd-master
cmake -DCMATE_INSTALL_PREFIX=/usr/local/lsyncd
make && make install
```

## 配置

### 配置文件示例

1. 创建配置文件

```shell
touch lsync.conf
```

2. 远程文件拷贝：编辑配置文件（完整）

```lua
-- User configuration file for lsyncd.
-- Simple example for default rsync, but executing moves through on the target.
-- For more examples, see /usr/share/doc/lsyncd*/examples/
settings {
   logfile    = "/tmp/auction_lsyncd.log",
   statusFile = "/tmp/auction_lsyncd.status",
   insist = true,
   statusInterval = 10,
   inotifyMode = "CloseWrite",
   nodaemon = true,
   maxProcesses = 10,
}
sync {
   default.rsyncssh,
   source="/files",
   host="192.168.1.9",
   targetdir="/disk/files",
   rsync = {
         binary = "/usr/bin/rsync",
     	 archive = true,
     	 compress = false,
         verbose = true,
     	 whole_file = false
   },
   ssh = {
     port = 22
   }
}
```

3. 本地目录同步cp

```lua
sync {
    default.direct,
    source    = "/tmp/src",
    target    = "/tmp/dest",
    delay = 1
    maxProcesses = 1
}
```

4. 本地目录同步rsync

```lua
sync {
    default.rsync,
    source    = "/tmp/src",
    target    = "/tmp/dest",
    excludeFrom = "/etc/rsyncd.d/rsync_exclude.lst",
    rsync     = {
        binary = "/usr/bin/rsync",
        archive = true,
        compress = true,
        bwlimit   = 2000
    } 
}
```

### 选项说明

1. settings

* logfile 定义日志文件
* statusFile 定义状态文件
* nodaemon=true 表示不启用守护模式，默认
* statusInterval 将lsyncd的状态写入上面的statusFile的间隔，默认10秒
* inotifyMode 指定inotify监控的事件，默认是CloseWrite,还可以是Modify或CloseWrite or Modify
* maxProcesses 同步进程的最大个数
* maxDelay 累计到多少所监控的事件激活一次同步，即使后面的delay延迟时间还未到

2. sync

运行模式：rsync,rsyncssh,direct
* default.sync 本地间同步，使用rsync
* default.direct 本地间同步，使用cp
* default.rsyncssh 同步到远程主机目录，rsync的ssh模式，需要使用key来认证
其他
* source 同步的源目录，使用绝对路径
* target 定义目的地址
    * /tmp/dest ：本地目录同步，可用于direct和rsync模式
    * 172.29.88.223:/tmp/dest ：同步到远程服务器目录，可用于rsync和rsyncssh模式，拼接的命令类似于/usr/bin/rsync -ltsd –delete –include-from=- –exclude=* SOURCE TARGET，剩下的就是rsync的内容了，比如指定username，免密码同步
    * 172.29.88.223::module ：同步到远程服务器目录，用于rsync模式

3. rsync
* bwlimit 限速，单位kb/s
* compress 压缩传输。默认true
* perms 默认保留文件权限
* host 主机地址

## 启动lsyncd

```shell
lsyncd lsyncd.conf >> /dev/null 2>&1
```

## lsyncd其他功能

可监控目录下的文件，根据触发的时间自己定义要执行的命令