# FastDFS 安装记录

## 安装环境描述

|主机地址|安装软件列表|
|---|---|
|172.18.255.20|tracker、libfastcommon、nginx|
|172.18.255.21|tracker、libfastcommon、nginx|

## 通用安装包下载及安装

安装fastdfs，nginx等需要编译工具：gcc gcc-c++

```shell
yum install -y epel-release
yum install -y gcc gcc-c++
# 安装nginx过程中需要安装的资源包
yum install pcre-devel openssl openssl-devel libxml2 libxml2-dev libxslt-devel gd gd-devel perl-devel perl-ExtUtils-Embed gperftools
```

## FastDFS环境下载及安装

### 1. 安装[libfastcommon](https://github.com/happyfish100/libfastcommon/archive/V1.0.48.tar.gz)

```shell
wget https://github.com/happyfish100/libfastcommon/archive/V1.0.48.tar.gz
tar -zxvf V1.0.48.tar.gz -C /usr/local/
cd /usr/local/libfastcommon-1.0.48
./make.sh
./make.sh install
```
### 2. 安装[tracker](https://github.com/happyfish100/fastdfs/archive/V6.07.tar.gz)

```shell
wget https://github.com/happyfish100/fastdfs/archive/V6.07.tar.gz
tar -zxvf V6.07.tar.gz -C /usr/local/
cd /usr/local/fastdfs-6.0.7
./make.sh
./make.sh install
```

### 3. 安装[nginx-module](https://github.com/happyfish100/fastdfs-nginx-module/archive/V1.22.tar.gz)

> 下载`nginx`之前需确认`nginx`版本，使用命令为`nginx -V`
> 确认完`nginx`版本后下载对应版本资源

```shell
wget https://github.com/happyfish100/fastdfs-nginx-module/archive/V1.22.tar.gz
wget http://nginx.org/download/nginx-1.16.1.tar.gz
tar -zxvf V1.22.tar.gz
tar -zxvf nginx-1.16.1.tar.gz
```

编译安装nginx

```shell
./configure --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/tmp/client_body --http-proxy-temp-path=/var/lib/nginx/tmp/proxy --http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi --http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi --http-scgi-temp-path=/var/lib/nginx/tmp/scgi --pid-path=/run/nginx.pid --lock-path=/run/lock/subsys/nginx --user=nginx --group=nginx --with-file-aio --with-ipv6 --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-stream_ssl_preread_module --with-http_addition_module --with-http_xslt_module=dynamic --with-http_image_filter_module=dynamic --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_degradation_module --with-http_slice_module --with-http_stub_status_module --with-http_perl_module=dynamic --with-http_auth_request_module --with-mail=dynamic --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream=dynamic --with-stream_ssl_module --with-google_perftools_module --with-debug --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --add-module=../fastdfs-nginx-module-1.22/src
```

### 4. 安装`libevent`

```shell
yum install -y libevent libevent-devel
```

### 5. 安装[berkeley-db](http://www.oracle.com/technology/software/products/berkeley-db/index.html)，给FastDHT使用

```shell
# 下载需登陆
wget https://download.oracle.com/otn/berkeley-db/db-18.1.40.tar.gz?AuthParam=1614044387_2f0093cb8050ae35ab0bdd812d391ed7 -O db-18.1.40.tar.gz
tar -zxvf db-18.1.40.tar.gz -C /usr/local/src/
cd /usr/local/src/db-18.1.40/
mkdir ../docs/bdb-sql
mkdir ../docs/gsg_db_server
cd build_unix/
../dist/configure --prefix=/usr/local/db-18.1.40
make
make install
ln -s /usr/local/db-18.1.40/lib/libdb-18.so /usr/lib/
ln -s /usr/local/db-18.1.40/lib/libdb-18.so /usr/lib64/
ln -s /usr/local/db-18.1.40/lib/libdb-18.1.so /usr/lib/
ln -s /usr/local/db-18.1.40/lib/libdb-18.1.so /usr/lib64/
```

### 6. 安装[FastDHT](https://github.com/happyfish100/fastdht/archive/master.zip)

```shell
wget https://github.com/happyfish100/fastdht/archive/master.zip -O fastdht-master.zip
unzip hastdht-master.zip
cd fastdht-master/

# 修改配置文件
vim make.sh
# 该句内容
CFLAGS='-Wall -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE' 
# 改为
CFLAGS='-Wall -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE -I/usr/local/db-18.1.40/include -L/usr/local/db-18.1.40/lib'

# 保存退出后
./make.sh
./make.sh install
mkdir /disk/disk1/home/fastdht
```

## 配置文件

### 1. 拷贝相关配置到`/etc/fdfs`下

   * 拷贝fastdfs相关配置

   ```shell
   cd /usr/local/fastdfs-6.0.7
   cp conf/* /etc/fdfs/
   ```

   * 拷贝nginx-module

   ```shell
   cd ~/fastdfs-nginx-module-1.22/src/
   cp mod_fastdfs.conf /etc/fdfs
   ```

### 2. 修改相关配置文件

#### * tracker.conf

```shell
base_path = /disk/disk1/home/fastdfs
http.server_port = 80
```

#### * storage.conf

```shell
# 这里需要注意的是，group_name命名不能含有-或_
group_name = yc
base_path = /disk/disk1/home/fastdfs
store_path0 = /disk/disk1/fdfs_storage
tracker_server = 172.18.255.20:22122
tracker_server = 172.18.255.21:22122
http.server_port = 88
```

#### * client.conf

```shell
base_path = /disk/disk1/home/fastdfs
tracker_server = 172.18.255.20:22122
tracker_server = 172.18.255.21:22122
```

#### * mod_fastdfs.conf

```shell
base_path = /disk/disk1/home/fastdfs
tracker_server = 172.18.255.20:22122
tracker_server = 172.18.255.21:22122
group_name = yc
url_have_group_name = true
```

### 3. nginx相关配置

修改主配置

```shell
cd /etc/nginx
vim nginx.conf

# 注释nginx的默认访问server
#    server {
#        listen       80 default_server;
#        listen       [::]:80 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#        location = /404.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#        location = /50x.html {
#        }
#    }
```

添加fdfs server

```shell
cd conf.d
vim fdfs.conf

#### 增加的内容

server {
    listen 80;
    server_name 172.18.255.21;
    location /yc/ {
        # root /disk/disk1/fdfs_storage/data;
        ngx_fastdfs_module;
    }
    access_log /disk/disk1/nginx/logs/access_fdfs.log;
    error_log /disk/disk1/nginx/logs/error_fdfs.log;
}
```

### 4. FastDHT配置

A `/etc/fdht`

#### * fdht_client.conf
```shell
base_path=/disk/disk1/home/fastdht
#include /etc/fdht/fdht_servers.conf
```
#### * fdht_servers.conf
```shell
group_count = 1
group0 = 172.18.255.20:11411
group0 = 172.18.255.21:11411
```
#### * fdhtd.conf
```shell
base_path=/disk/disk1/home/fastdht
#include /etc/fdht/fdht_servers.conf
```

B `/etc/fdfs`

#### * storage.conf

```shell
check_file_duplicate = 1
key_namespace = FastDFS
keep_alive = 1
#include /etc/fdht/fdht_servers.conf
```

## 多group配置

### 1. 复制一份`storage.conf`到`storage_test.conf`并修改配置

```shell
port = 23001
base_path = /disk/disk1/home/fastdfs_test
store_path_count = 1
store_path0 = /disk/disk8/fdfs_storage_test
key_namespace = FastDFSTest # 需要注意的地方
http.server_port = 89
```

### 2. 启动新的storage

```shell
fdfs_storaged /etc/fdfs/storage_test.conf restart
```

### 3. 修改配置`mod_fastdfs.conf`

```shell
#storage_server_port=23000
group_name=yc/test
#store_path_count=8
#store_path0=/disk/disk1/fdfs_storage
#store_path1=/disk/disk2/fdfs_storage
#store_path2=/disk/disk3/fdfs_storage
#store_path3=/disk/disk4/fdfs_storage
#store_path4=/disk/disk5/fdfs_storage
#store_path5=/disk/disk6/fdfs_storage
#store_path6=/disk/disk7/fdfs_storage
#store_path7=/disk/disk8/fdfs_storage
group_count = 2
[group1]
group_name=yc
storage_server_port=23000
store_path_count=8
store_path0=/disk/disk1/fdfs_storage
store_path1=/disk/disk2/fdfs_storage
store_path2=/disk/disk3/fdfs_storage
store_path3=/disk/disk4/fdfs_storage
store_path4=/disk/disk5/fdfs_storage
store_path5=/disk/disk6/fdfs_storage
store_path6=/disk/disk7/fdfs_storage
store_path7=/disk/disk8/fdfs_storage
[group2]
group_name=test
storage_server_port=23001
store_path_count=1
store_path0=/disk/disk8/fdfs_storage_test
```

### 4. 修改`nginx`配置文件并重启`nginx`

```shell
########
vim /etc/nginx/conf.d/fdfs.conf
location /test/ {
#    root /disk/disk1/fdfs_storage/data;
	ngx_fastdfs_module;
}
######## 保存退出
######## 重启nginx
systemctl restart nginx
```

## 防火墙

```shell
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --add-port=88/tcp --permanent
firewall-cmd --add-port=22122/tcp --permanent
firewall-cmd --add-port=23000/tcp --permanent
firewall-cmd --add-port=89/tcp --permanent
firewall-cmd --add-port=23001/tcp --permanent
firewall-cmd --reload
```

## 启动及设置开机启动

```shell
# 启动
systemctl start nginx
systemctl start fdfs_trackerd
systemctl start fdfs_storaged
/usr/local/bin/fdhtd /etc/fdht/fdhtd.conf
# 开机启动
systemctl enable nginx
systemctl enable fdfs_trackerd
systemctl enable fdfs_storaged
#####
vim /etc/rc.local
## 增加
/usr/local/bin/fdhtd /etc/fdht/fdhtd.conf
## 保存后退出
chmod +x /etc/rc.local
```

## 问题排查

1. `nginx`本身错误及访问错误

具体问题具体分析

2. fdfs启动后错误

可到`base_path`中，找到logs对应的文件，查看错误，并做对应解决处理

3. 测试`FastDHT`是否安装正确

```shell
fdht_set /etc/fdht/fdht_client.conf ceshi:user001 name='ceshi',age=18;
fdht_get /etc/fdht/fdht_client.conf ceshi:user001 nanme,age;
fdht_delete /etc/fdht/fdht_client.conf ceshi:user001 name;
fdht_delete /etc/fdht/fdht_client.conf ceshi:user001 age;
```

## 测试是否成功

* 上传文件
```shell
fdfs_test /etc/fdfs/client.conf upload test.jpg
```

* 浏览器访问

http://172.18.255.20/yc/M00...jpg
http://172.18.255.21/yc/M00...jpg

