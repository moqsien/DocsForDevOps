# netdata安装教程

1. 安装依赖

```shell
yum install autoconf automake curl gcc git libmnl-devel libuuid-devel openssl-devel libuv-devel lz4-devel Judy-devel make nc pkgconfig python zlib-devel
```

2. 下载资源

```shell
git clone https://github.com/netdata/netdata.git --depth=100
```

3. 安装

```shell
cd netdata
./netdata-installer.sh
```

4. 设置开机启动

```shell
systemctl enable netdata
```

5. 开启防火墙

```
firewall-cmd --add-port=19999/tcp --permernent
firewall-cmd --reload
```

6. 设置记录保留时间为1天

```shell
cd /etc/netdata
vim netdata.conf

# 20行位置修改[global]下的history = 86400
```

