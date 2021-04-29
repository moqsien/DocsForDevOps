# docker-compose安装kafka集群

## 0 安装docker（略）

## 1. 安装docker-compose

1. 下载安装文件

```shell
curl -L https://github.com/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

2. 添加权限

```shell
chmod +x /usr/local/bin/docker-compose
```

3. 查看版本

```shell
docker-compose --version
```

## 2. 安装zookeeper及kafka镜像

1. 查看镜像

```shell
docker search zookeeper
docker search kafka
```

2. 下载镜像

```shell
docker pull wurstmeister/zookeeper
docker pull wurstmeister/kafka
docker pull sheepkiller/kafka-manager #管理工具
```

## 3. 创建必要文件及文件夹（docker-compose.yml同一目录下）

1. kafka文件夹

```shell
mkdir kafka1
mkdir kafka2
mkdir kafka3
```

2. zookeeper文件夹

```shell
mkdir zookeeper1
mkdir zookeeper2
mkdir zookeeper3
```

3. zookeeper配置文件

```shell
mkdir zooConfig
cd zooConfig
mkdir zoo1
mkdir zoo2
mkdir zoo3
```

4. 在zoo1,zoo2,zoo3中分别创建myid文件，并写入分别写入id数字，如zoo1中的myid中写入1

5. 创建zoo配置文件zoo.cfg

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
dataDir=/data
dataLogDir=/datalog
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
autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
autopurge.purgeInterval=1
server.1= 172.23.0.11:2888:3888
server.2= 172.23.0.12:2888:3888
server.3= 172.23.0.13:2888:3888
```

## 4. 创建网络

```shell
docker network create --driver bridge --subnet 172.23.0.0/25 --gateway 172.23.0.1  zookeeper_network
```

## 4. 创建docker-compose.yml文件

```yml
version: '2'

services:

  zoo1:
    image: zookeeper:3.4.14
    restart: always
    container_name: zoo1
    hostname: zoo1
    ports:
    - "2181:2181"
    volumes:
    - "./zooConfig/zoo.cfg:/conf/zoo.cfg"
    - "/disk/docker/zookeeper1/data:/data"
    - "/disk/docker/zookeeper1/datalog:/datalog"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    networks:
      default:
        ipv4_address: 172.23.0.11

  zoo2:
    image: zookeeper:3.4.14
    restart: always
    container_name: zoo2
    hostname: zoo2
    ports:
    - "2182:2181"
    volumes:
    - "./zooConfig/zoo.cfg:/conf/zoo.cfg"
    - "/disk/docker/zookeeper2/data:/data"
    - "/disk/docker/zookeeper2/datalog:/datalog"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    networks:
      default:
        ipv4_address: 172.23.0.12

  zoo3:
    image: zookeeper:3.4.14
    restart: always
    container_name: zoo3
    hostname: zoo3
    ports:
    - "2183:2181"
    volumes:
    - "./zooConfig/zoo.cfg:/conf/zoo.cfg"
    - "/disk/docker/zookeeper3/data:/data"
    - "/disk/docker/zookeeper3/datalog:/datalog"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    networks:
      default:
        ipv4_address: 172.23.0.13

  kafka1:
    image: wurstmeister/kafka:2.12-2.0.1
    restart: always
    container_name: kafka1
    hostname: kafka1
    ports:
    - 9092:9092
    - 9999:9999
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://172.18.255.9:9092
      KAFKA_ADVERTISED_HOST_NAME: kafka1
      KAFKA_HOST_NAME: kafka1
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_BROKER_ID: 0
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      JMX_PORT: 9999
    volumes:
    - /etc/localtime:/etc/localtime
    - "/disk/docker/kafka1/logs:/kafka"
    links:
    - zoo1
    - zoo2
    - zoo3
    networks:
      default:
        ipv4_address: 172.23.0.14

  kafka2:
    image: wurstmeister/kafka:2.12-2.0.1
    restart: always
    container_name: kafka2
    hostname: kafka2
    ports:
    - 9093:9092
    - 9998:9999
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://172.18.255.9:9093
      KAFKA_ADVERTISED_HOST_NAME: kafka2
      KAFKA_HOST_NAME: kafka2
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
      KAFKA_ADVERTISED_PORT: 9093
      KAFKA_BROKER_ID: 1
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      JMX_PORT: 9999
    volumes:
    - /etc/localtime:/etc/localtime
    - "/disk/docker/kafka2/logs:/kafka"
    links:
    - zoo1
    - zoo2
    - zoo3
    networks:
      default:
        ipv4_address: 172.23.0.15

  kafka3:
    image: wurstmeister/kafka:2.12-2.0.1
    restart: always
    container_name: kafka3
    hostname: kafka3
    ports:
    - 9094:9092
    - 9997:9999
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://172.18.255.9:9094
      KAFKA_ADVERTISED_HOST_NAME: kafka3
      KAFKA_HOST_NAME: kafka3
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
      KAFKA_ADVERTISED_PORT: 9094
      KAFKA_BROKER_ID: 2
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      JMX_PORT: 9999
    volumes:
    - /etc/localtime:/etc/localtime
    - "/disk/docker/kafka3/logs:/kafka"
    links:
    - zoo1
    - zoo2
    - zoo3
    networks:
      default:
        ipv4_address: 172.23.0.16

  kafka-manager:
    image: hlebalbau/kafka-manager:1.3.3.22
    restart: always
    container_name: kafka-manager
    hostname: kafka-manager
    ports:
    - 9000:9000
    links:
    - kafka1
    - kafka2
    - kafka3
    - zoo1
    - zoo2
    - zoo3
    environment:
      ZK_HOSTS: zoo1:2181,zoo2:2181,zoo3:2181
      KAFKA_BROKERS: kafka1:9092,kafka2:9093,kafka3:9094
      APPLICATION_SECRET: letmein
      KAFKA_MANAGER_AUTH_ENABLED: "true"
      KAFKA_MANAGER_USERNAME: "admin"
      KAFKA_MANAGER_PASSWORD: "admin"
      KM_ARGS: -Djava.net.preferIPv4Stack=true
    networks:
      default:
        ipv4_address: 172.23.0.10

networks:
  default:
    external:
      name: zookeeper_network
```
## 6. 启停集群

1. 启动集群

```shell
docker-compose -f docker-compose.yml up -d
```

2. 停止集群

```shell
docker-compose -f docker-compose.yml stop
```

3. 单个节点停止

```shell
docker rm -f zoo1
```

## 7. 查看zookeeper集群是否正常

```shell
docker exec -it zoo1 bash
bin/zkServer.sh status # mode 为leader或follower正常
```

## 8. 创建topic

1. 验证，每个list理论上都可以看到新建的topic

```shell
docker exec -it kafka1 bash
kafka-topics.sh --create --zookeeper zoo1:2181 --replication-factor 1 --partitions 3 --topic test001
kafka-topics.sh --list --zookeeper zoo1:2181
kafka-topics.sh --list --zookeeper zoo2:2181
kafka-topics.sh --list --zookeeper zoo3:2181
```

2. 生产消息

```shell
kafka-console-producer.sh --broker-list kafka1:9092,kafka2:9093,kafka3:9094 --topic test001
```

3. 消费消息

```shell
kafka-console-consumer.sh --bootstrap-server kafka1:9092,kafka2:9093,kafka3:9094 --topic test001 --from-beginning
```

## 9. 防火墙开启相关端口

```shell
firewall-cmd --add-ports=9000/tcp --permernent
firewall-cmd --reload
```