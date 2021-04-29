# ES相关操作详细文档

## ES的安装

### 所需安装软件及插件

|名称|下载地址|
|---|---|
|jdk-8u191|[下载地址](https://download.oracle.com/otn-pub/java/jdk/8u191-b12/2787e4a523244c269598db4e85c51e0c/jdk-8u191-linux-x64.tar.gz)|
|elasticsearch-6.5.4|[下载地址](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.4.tar.gz)|
|kibana-6.5.4|[下载地址](https://artifacts.elastic.co/downloads/kibana/kibana-6.5.4-linux-x86_64.tar.gz)|
|logstash-6.5.4|[下载地址](https://artifacts.elastic.co/downloads/logstash/logstash-6.5.4.tar.gz)|
|elasticsearch-analysis-ik-6.5.4|[下载地址](https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.5.4/elasticsearch-analysis-ik-6.5.4.zip)|

### 安装过程

```shell
cd /tmp
wget https://download.oracle.com/otn-pub/java/jdk/8u191-b12/2787e4a523244c269598db4e85c51e0c/jdk-8u191-linux-x64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.4.tar.gz
wget https://artifacts.elastic.co/downloads/kibana/kibana-6.5.4-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/logstash/logstash-6.5.4.tar.gz
wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.5.4/elasticsearch-analysis-ik-6.5.4.zip
su -
mkdir /disk/elasticsearch
chown -R work.work /disk/elasticsearch
exit
tar -zxvf jdk-8u191-linux-x64.tar.gz -C /usr/local/jdk
tar -zxvf elasticsearch-6.5.4.tar.gz -C /disk/elasticsearch
tar -zxvf kibana-6.5.4-linux-x86_64.tar.gz -C /disk/elasticsearch
tar -zxvf logstash-6.5.4.tar.gz -C /disk/elasticsearch
cd /disk/elasticsearch/elasticsearch-6.5.4/plugins
mkdir analysis-ik
cd analysis-ik
cp /tmp/elasticsearch-analysis-ik-6.5.4.zip ./
unzip elasticsearch-analysis-id-6.5.4.zip
rm elasticsearch-analysis-id-6.5.4.zip
```

### JDK环境变量配置

```shell
vim /etc/profile

	# 在文件末尾添加
    export JAVA_HOME=/usr/local/jdk/jdk1.8.0_191
    export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$JAVA_HOME/lib/tools.jar
    export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
	# 保存后退出
	
source /etc/profile
```

### elasticsearch配置

```shell
cd /disk/elasticsearch/elasticsearch-6.5.4/config
vim elasticsearch.yml

    # 修改相关配置
    cluster.name: elasticsearch # 集群名称
    node.name: online-x # x代表编号
    bootstrap.memory_lock: false
    bootstrap.system_call_filter: false
    path.data: /disk/elasticsearch/data/data # 数据保存位置
    path.logs: /disk/elasticsearch/data/logs # 日志保存位置
    network.host: 0.0.0.0
    http.port: 9200
    # 保存后退出
    
mkdir /disk/elasticsearch/data/data
mkdir /disk/elaticsearch/data/logs
vim jvm.options

    # 修改相关配置
    -Xms32g
    -Xmx32g
    # 保存后退出
    
```

### kibana配置

```shell
cd /disk/elasticsearch/kibana-6.5.4-linux-x86_64/config
vim kibana.yml

	# 修改相关配置
	server.port: 5602
	server.host: "localhost"
	elasticsearch.url: "http://localhost:9200"
	# 保存后推出
	
```

### elasticsearch启动

```shell
cd /disk/elasticsearch/elasticsearch-6.5.4
bin/elasticsearch
```

**报错解决方法**

```shell
vim /etc/security/limits.conf

	# 修改
	* soft nofile 65536
    * hard nofile 65536
    # 修改后保存
    
vim /etc/sysctl.conf
	
	# 修改
	vm.max_map_count=655360
	# 修改后保存

sysctl -p
vim /etc/sysconfig/selinux
	
	# 修改
	SELINUX=disabled
	# 修改后保存

```

**后台启动**

```shell
cd /disk/elasticsearch/elasticsearch-6.5.4
bin/elasticsearch -d
```

### kibana启动

```shell
cd /disk/elasticsearch/kibana-6.5.4-linux-x86_64
nohup bin/kibana &
```

**nginx代理，添加密码验证**

```shell
su -
cd /etc/nginx/conf.d
touch kibana.conf
vim kibana.conf
	#文件内容
	upstream kibana_server {
        server localhost:5602;
    }
    server
    {
        listen 5601;
        server_name _;
        auth_basic "Kibana Auth";
        auth_basic_user_file /etc/nginx/passwd/kibana.passwd;
        proxy_redirect http:// $scheme://;
        port_in_redirect on;
        location / {
            proxy_pass http://kibana_server;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Cookie $http_cookie;
        }
        access_log /disk/nginx/logs/access_kibana.log;
    }
    # 文件内容止

mkdir /etc/nginx/passwd
htpasswd -c -b /etc/nginx/passwd/kibana.passwd admin youcheng2017
nginx -t #没有错误后重启nginx，access_log文件夹路径需存在
service nginx restart
```
### 防火墙配置

```shell
firewall-cmd --zone=public --add-port=9200/tcp --permanent
firewall-cmd --zone=public --add-port=9300/tcp --permanent
firewall-cmd --zone=public --add-port=5601/tcp --permanent
firewall-cmd --reload
```