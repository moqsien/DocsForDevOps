# zookeeper安装

1. 下载bin文件

```shell
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.5.6/apache-zookeeper-3.5.6-bin.tar.gz
```

2. 解压到/usr/local下

```shell
tar -zxvf apache-zookeeper-3.5.6-bin.tar.gz -C /usr/local
```

3. 修改配置文件

```shell
cd /usr/local
ln -s apache-zookeeper-3.5.6-bin zookeeper
cd zookeeper/conf
cp zoo_sample.cfg zoo.cfg
```

配置文件内容：

```shell
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/kafka/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

server.1=172.18.255.7:2888:3888
server.2=172.18.255.8:2888:3888
server.3=172.18.255.9:2888:3888
```

在dataDir中添加myid，对应ID填入server.{x}对应值

4. 启动

```shell
cd /usr/local/zookeeper/bin
./zkServer.sh start
```

5. 查看是否成功

```shell
./zkServer.sh status
```

