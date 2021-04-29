# 搭建zabbix监控平台

## 搭建LNMP环境

### 安装PHP

1. 安装基础模块

```shell
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

yum install -y php56w php56w-opcache php56w-bcmath php56w-common php56w-gd php56w-mbstring php56w-mcrypt php56w-mysqlnd php56w-process php56w-fpm php56w-pdo php56w-xmlrpc php56w-cli php56w-xml php56w-xmlreader php56w-xmlwriter php56w-net-socket php56w-ctype php56w-session php56w-gettext php56w-ldap
```

2. 修改配置

> 将user和group改为work
> 

```shell
vim /etc/php-fpm.d/www.conf
```

3. 修改session文件权限

```shell
chown work.work /var/lib/php/session
```

4. 启动与添加开机自启

```shell
systemctl start php-fpm.service

systemctl enable php-fpm.service
```

### 创建数据库

1. 新建数据库并配置账号密码

```shell
create database zabbix character set utf8 collate utf8_bin;
grant all privileges on zabbix.* to 'zabbix'@'%' identified by 'Zabbix@2020';
```

## 搭建zabbix相关服务

### 安装zabbix-server，参考[地址](https://www.zabbix.com/documentation/4.2/manual/installation/install_from_packages/rhel_centos)

1. 安装基本的源信息，参考[地址](https://www.zabbix.com/documentation/4.2/manual/installation/install_from_packages/rhel_centos)

> 需要安装zabbix-agent的主机需要安装rpm源
>
> rpm -Uvh https://repo.zabbix.com/zabbix/4.2/rhel/7/x86_64/zabbix-release-4.2-2.el7.noarch.rpm

```shell
rpm -Uvh https://repo.zabbix.com/zabbix/4.2/rhel/7/x86_64/zabbix-release-4.2-2.el7.noarch.rpm
yum-config-manager --enable rhel-7-server-optional-rpms

1. yum provides '*/applydeltarpm'
2. yum install -y deltarpm
```
2. 安装zabbix_server

```shell
yum install zabbix-server-mysql zabbix-web-mysql zabbix-proxy-mysql
```

3. 修改server配置
```shell
vim /etc/zabbix/zabbix_server.conf
```

> 修改必要的数据库配置为上文中新创建的数据库信息
>
> 监控发送信息脚本：/usr/lib/zabbix/alertscripts
>
> 主程序位置：/usr/share/zabbix

4. 配置nginx:

```shell
vim /etc/nginx/conf.d/zabbix.conf
#>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
server {
    listen      80;
    server_name zabbix.yczcjk.com;
    location / {
        root /usr/share/zabbix;
        index  index.php;
#        if (!-e $request_filename) {
#            rewrite ^/(.*)$ /index.php/$1 last;
#        }
    }
    location ~ ^/.+\.php {
        root /usr/share/zabbix;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        fastcgi_index  index.php;
        fastcgi_split_path_info ^(.+\.php)(/?.+)$;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param PATH_TRANSLATED $document_root$fastcgi_path_info;
        include        fastcgi_params;
        fastcgi_pass   127.0.0.1:9000;
    }
    access_log /disk/web/nginx/logs/zabbix_access.log;
    error_log /disk/web/nginx/logs/zabbix_error.log;
}
```

5. 导入数据库文件

```
zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix

zcat /usr/share/doc/zabbix-proxy-mysql*/schema.sql.gz | mysql -uzabbix -p zabbix_proxy
```

6. 启动zabbix_server

```shell
systemctl start zabbix-server.service
systemctl enable zabbix-server.service
```

7. 登录浏览器进行相关配置

> http://zabbix.yczcjk.com
> 
8. 增加钉钉脚本文件

```shell
cd /usr/lib/zabbix/alertscripts

vim ding.py
#>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
#!/usr/bin/python
# -*- coding: utf-8 -*-
import requests
import json
import sys
import os
headers = {'Content-Type': 'application/json;charset=utf-8'}
#api_url后跟告警机器人的webhook
api_url = "https://oapi.dingtalk.com/robot/send?access_token=1cc14429b8453506f6a358cf0005cb005561f159a8f43d8317608c1005a16453"
def msg(text):
   json_text= {
    "msgtype": "text",
    "text": {
        "content": text
    }
   }
   print(requests.post(api_url,json.dumps(json_text),headers=headers).content)
if __name__ == '__main__':
   text = sys.argv[1]
   msg(text)
```
增加相关权限

```shell
chmod +x ding.py
```

### 安装net-snmp，用于测试snmpV2相关协议

```shell
yum -y install net-snmp-devel net-snmp-utils
```

### 安装zabbix-agent

> 每一个需要监控的主机都需要安装该应用
>
> 需要安装源：rpm -Uvh https://repo.zabbix.com/zabbix/4.2/rhel/7/x86_64/zabbix-release-4.2-2.el7.noarch.rpm

1. 安装基础应用

```shell
yum install zabbix-agent
```

2. 修改相关配置

```shell
vim /etc/zabbix/zabbix_agentd.conf

Server=172.18.255.8 # 服务器地址
ServerActive=172.18.255.8 # 服务器地址
Hostname=172.18.255.8 # 当前主机名称，不同主机不同
```

4. 启动与自启

```shell
systemctl start zabbix-agent.service
systemctl enable zabbix-agent.service
```

## 戴尔服务器硬件监控相关组件安装

### centos7.6安装omsa

可对戴尔服务器硬件监控，参考[地址](https://www.dell.com/support/manuals/cn/zh/cndhs1/dell-opnmang-srvr-admin-v7.4/omsa_cli-v3/omreport-storage-命令?guid=guid-3d33a63a-4b6d-4e01-b8aa-9f5f2f00f200&lang=zh-cn)

1. 加载戴尔数据源

```shell
wget -q -O - http://linux.dell.com/repo/hardware/latest/bootstrap.cgi | bash
```

2. 安装相关程序

```shell
yum install -y srvadmin-all

ln -s /opt/dell/srvadmin/sbin/omconfig /usr/local/bin/
```

3. 禁用某些项

```shell
echo "/usr/bin/omconfig system webserver action=stop" >> /opt/dell/srvadmin/sbin/srvadmin-services.sh
```

4. 启动监控程序并开机自启

```shell
/opt/dell/srvadmin/sbin/srvadmin-services.sh start

vim /etc/rc.local
#>>>>>>>>>>>>>>>>文件末尾添加
/opt/dell/srvadmin/sbin/srvadmin-services.sh start
```

5. 修改zabbix-agent相关配置，并重启zabbix-agent

```shell
vim /etc/zabbix/zabbix_agentd.d/userparameter_hardware.conf
#>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
# 电池健康
UserParameter=hardcheck.hardware_battery,omreport chassis batteries|awk '/^Status/{if($NF=="Ok") {print 1} else {print 0}}'
# 风扇健康
UserParameter=hardcheck.hardware_fan_health,awk -vhardware_fan_number=`omreport chassis fans|grep -c "^Index"` -vhardware_fan=`omreport chassis fans|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` 'BEGIN{if(hardware_fan_number==hardware_fan) {print 1} else {print 0}}'
# 内存健康
UserParameter=hardcheck.hardware_memory_health,awk -vhardware_memory=`omreport chassis memory|awk '/^Health/{print $NF}'` 'BEGIN{if(hardware_memory=="Ok") {print 1} else {print 0}}'
# UserParameter=hardcheck.hardware_nic_health,awk -vhardware_nic_number=`omreport chassis nics |grep -c "Interface Name"` -vhardware_nic=`omreport chassis nics |awk '/^Connection Status/{print $NF}'|wc -l` 'BEGIN{if(hardware_nic_number==hardware_nic) {print 1} else {print 0}}'
# CPU健康
UserParameter=hardcheck.hardware_cpu,omreport chassis processors|awk '/^Health/{if($NF=="Ok") {print 1} else {print 0}}'
# 电源监控
UserParameter=hardcheck.hardware_power_health,awk -vhardware_power_number=`omreport chassis pwrsupplies|grep -c "Index"` -vhardware_power=`omreport chassis pwrsupplies|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` 'BEGIN{if(hardware_power_number==hardware_power) {print 1} else {print 0}}'
# UserParameter=hardcheck.hardware_temp,omreport chassis temps|awk '/^Status/{if($NF=="Ok") {print 1} else {print 0}}'|head -n 1
# 硬件温度
UserParameter=hardcheck.hardware_temp,awk -vhardware_temp=`omreport chassis temps|awk '/^Main/{print $NF}'` 'BEGIN{if(hardware_temp=="Ok") {print 1} else {print 0}}'
# 磁盘健康
UserParameter=hardcheck.hardware_physics_health,awk -vhardware_physics_disk_number=`omreport storage pdisk controller=0|grep -c "^ID"` -vhardware_physics_disk=`omreport storage pdisk controller=0|awk '/^Failure Predicted/{if($NF=="No") count+=1}END{print count}'` 'BEGIN{if(hardware_physics_disk_number==hardware_physics_disk) {print 1} else {print 0}}'
# 虚拟磁盘健康
UserParameter=hardcheck.hardware_virtual_health,awk -vhardware_virtual_disk_number=`omreport storage vdisk controller=0|grep -c "^ID"` -vhardware_virtual_disk=`omreport storage vdisk controller=0|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` 'BEGIN{if(hardware_virtual_disk_number==hardware_virtual_disk) {print 1} else {print 0}}'
# UserParameter=check.zichan.system,/bin/cat /etc/redhat-release
# UserParameter=check.zichan.bmc_name,/bin/echo iDRAC_info
# UserParameter=check.dev[*],/bin/bash /Data/apps/zabbix/bin/custom/check-dev.sh $1 $2 $3
# cpu1温度
UserParameter=hardwaredetail.cpu1_temp,for i in {0..3} ; do if [[ `omreport chassis temps index=$i|grep "CPU1"` == *CPU1* ]]; then echo `omreport  chassis temps index=$i|grep Reading|cut -d ':' -f 2|awk '{if (NR==1) print $1}'`; fi; done
# cpu2温度
UserParameter=hardwaredetail.cpu2_temp,for i in {0..3} ; do if [[ `omreport chassis temps index=$i|grep "CPU2"` == *CPU2* ]]; then echo `omreport  chassis temps index=$i|grep Reading|cut -d ':' -f 2|awk '{if (NR==1) print $1}'`; fi; done
# 主板温度
UserParameter=hardwaredetail.system_board_inlet_temp,for i in {0..3} ; do if [[ `omreport chassis temps index=$i|grep "Inlet"` == *Inlet* ]]; then echo `omreport  chassis temps index=$i|grep Reading|cut -d ':' -f 2|awk '{if (NR==1) print $1}'`; fi; done
# 主板温度
UserParameter=hardwaredetail.system_board_exhaust_temp,for i in {0..3} ; do if [[ `omreport chassis temps index=$i|grep "Exhaust"` == *Exhaust* ]]; then echo `omreport  chassis temps index=$i|grep Reading|cut -d ':' -f 2|awk '{if (NR==1) print $1}'`; fi; done
# UserParameter=hardwaredetail.memory_detail_status,for ((i=0;i<`omreport chassis memory|grep "Slots Used"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'`;i++));do if [[ `omreport  chassis memory index=$i|grep "Status"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'` != "Ok" ]]; then echo `omreport  chassis memory index=$i|grep "Device Name"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'`" memory is bad; and errer code is "`omreport  chassis memory index=$i|grep "Failures"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'`;else echo "OK"; fi;done
# UserParameter=hardwaredetail.memory_detail_status,for i in `seq 0 $[$(omreport  chassis memory|grep "Slots Used"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g')-1]`;do if [[ `omreport  chassis memory index=$i|grep "Status"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'` != "Ok" ]]; then echo `omreport  chassis memory index=$i|grep "Device Name"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'`" memory is bad; and errer code is "`omreport  chassis memory index=$i|grep "Failures"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'`;else echo "OK"; fi;done
# 内存详情
UserParameter=hardwaredetail.memory_detail_status,if [[ `omreport chassis memory|awk '/^Health/{print $NF}'` =~ "Ok" ]];then echo "OK";else for i in `seq 0 $[$(omreport  chassis memory|grep "Slots Used"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g')-1]`;do if [[ `omreport  chassis memory index=$i|grep "Status"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'` != "Ok" ]]; then echo `omreport  chassis memory index=$i|grep "Device Name"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'`" memory is bad; and errer code is "`omreport  chassis memory index=$i|grep "Failures"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'`; fi;done;fi
# 日志详情
UserParameter=hardwaredetail.hardlog_detail_status,if [[ `omreport system esmlog|tail -n 4|grep "Severity"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'` != "Ok" ]];then echo `omreport system esmlog|tail -n 4`;else echo "OK";fi
# 日志详情
UserParameter=hardwaredetail.alertlog_detail_status,if [[ `omreport system alertlog|tail -n 6|grep "Severity"|awk -F: '{if (NR==1) print $2}'|sed 's/^[ \t]*//g'` != "Ok" ]];then echo `omreport system alertlog|tail -n 6`;else echo "OK";fi
# UserParameter=hardwaredetail.fan_fail_status,for i in `seq 0 $[$(omreport chassis fans|grep "Index"|wc -l)-1]` ; do if [[ `omreport  chassis fans index=$i|grep "Status"` != *Ok* ]]; then echo `omreport  chassis temps index=$i`;else echo "OK"; fi; done
# 风扇失败
UserParameter=hardwaredetail.fan_fail_status,if [[ `omreport chassis fans|grep -c "^Index"` == `omreport chassis fans|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` ]];then echo "OK";else for i in `seq 0 $[$(omreport chassis fans|grep "Index"|wc -l)-1]` ; do if [[ `omreport  chassis fans index=$i|grep "Status"` != *Ok* ]]; then echo `omreport  chassis temps index=$i`; fi; done; fi
# UserParameter=hardwaredetail.fan_fail_status,for ((i=0;i<`omreport chassis fans|grep "Index"|wc -l`;i++)) ; do if [[ `omreport  chassis fans index=$i|grep "Status"` != *Ok* ]]; then echo `omreport  chassis temps index=$i`;else echo "OK"; fi; done
#UserParameter=hardwaredetail.power_fail_status,for ((i=0;i<`omreport chassis pwrsupplies|grep "Index"|wc -l`;i++)) ; do if [[ `omreport  chassis pwrsupplies index=$i|grep "Status"` != *Ok* ]]; then echo `omreport  chassis pwrsupplies index=$i`;else echo "OK"; fi; done
# UserParameter=hardwaredetail.cpu_fail_status,for ((i=0;i<`omreport chassis processors|grep "Index"|wc -l`;i++)) ; do if [[ `omreport  chassis processors index=$i|grep "Status"` != *Ok* ]]; then echo `omreport  chassis processors index=$i`;else echo "OK"; fi; done
# UserParameter=hardwaredetail.temp_fail_status,for ((i=0;i<`omreport chassis temps|grep "Index"|wc -l`;i++)) ; do if [[ `omreport  chassis temps index=$i|grep "Status"` != *Ok* ]]; then echo `omreport  chassis temps index=$i`;else echo "OK"; fi; done
# 温度失败
UserParameter=hardwaredetail.temp_fail_status,if [[ `omreport chassis temps|awk '/^Main/{print $NF}'` == "Ok" ]];then echo "OK";else for ((i=0;i<`omreport chassis temps|grep "Index"|wc -l`;i++)) ; do if [[ `omreport  chassis temps index=$i|grep "Status"` != *Ok* ]]; then echo `omreport  chassis temps index=$i`; fi; done;fi
# UserParameter=hardwaredetail.disk_fail_status,if [[ `sudo /opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL -NoLog|grep "Firmware state"` =~ Online\,\ Spun\ Up|Hotspare\,\ Spun\ Up ]]; then echo "OK";else echo `sudo /opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL -NoLog|grep "Firmware state"|grep -v -E "Online, Spun Up|Hotspare, Spun Up"`;fi
```

### ubuntu16.04安装omsa

1. 安装相关源

```shell
wget -q -O - http://linux.dell.com/repo/hardware/latest/bootstrap.cgi | bash
echo 'deb http://linux.dell.com/repo/community/ubuntu xenial openmanage' | tee -a /etc/apt/sources.list.d/linux.dell.com.sources.list
gpg --keyserver pool.sks-keyservers.net --recv-key 1285491434D8786F
gpg -a --export 1285491434D8786F | sudo apt-key add -
apt-get update
apt-get install srvadmin-all
```

2. 其他配置同centos7.6

## 统一添加防火墙相关配置

### 防火墙端口

```shell
firewall-cmd --add-port=10050/tcp --permernant
firewall-cmd --reload
```

## 查看dell iDrac远程信息

1. 安装相关组件

```shell
yum install -y OpenIPMI ipmitool
```

2. 查看局域网信息

```shell
ipmitool lan print
```

## 其他问题

### 修改omsa使用信号量不足问题

1. 修改相关文件

```shell
vim /etc/sysctl.conf

#>>>>>>>>>>>>>>>>>>>
kernel.sem =1024 32000 100 2048
```

2. 使配置生效

```shell
sysctl -p
```

3. 查看是否生效

```shell
cat /proc/sys/kernel/sem
```

### 清空日志

1. 清除硬件日志、清除警报日志

```shell
omconfig system esmlog action=clear
omconfig system alertlog action=clear
```

## zabbix添加相关监控项

> 以下各项需在平台中自行探索添加

### 添加主机

### 添加用户群组

### 添加用户

### 添加报警媒介类型

### 添加模板

### 添加监控项

### 添加触发器

### 添加图形

### 创建钉钉动作

### 禁用guest用户