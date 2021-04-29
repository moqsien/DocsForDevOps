# 天眼查xtrabackup数据恢复注意事项

## 0. 首要步骤

1. 拿到移动硬盘后，两个解压缩文件放到服务器上，总计200G以上，放置位置/disk

	* 01_gongshang_hins6771473_data_20190207123924.tar.gz.zip
	* 02_lawsuit_hins6771477_data_20190207045505.tar.gz.zip

2. 解压缩这两个文件

	```shell
	unzip 01_gongshang_hins6771473_data_20190207123924.tar.gz.zip
	unzip 02_lawsuit_hins6771477_data_20190207045505.tar.gz.zip
	```
	> 解压过程中需要输入解压密码
	> 

## 1. 安装Percona-xtrabackup-24

1. 安装流程

	```shell
	# 安装源
	yum install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
	# 查看源信息
	yum list |grep percona
	# 安装
	yum install percona-xtrabackup-24
	```

## 2. 数据恢复

1. 使用rds_backup_extract.sh文件解压缩

	```shell
	mkdir gongshang
	bash rds_backup_extract.sh 01_gongshang_hins6771473_data_20190207123924.tar.gz -C gongshang
	mkdir lawsuit
	bash rds_backup_extract.sh 02_lawsuit_hins6771477_data_20190207045505.tar.gz -C lawsuit
	cp gongshang gongshang1 -r
	cp lawsuit lawsuit1 -r
	```
	
2. 对数据进行处理

	```shell
	innobackupex --defaults-file=/etc/my.cnf --apply-log --export /disk/gongshang
	innobackupex --defaults-file=gongshang1/backup-my.cnf --apply-log /disk/gongshang1
	innobackupex --defaults-file=/etc/my.cnf --apply-log --export /disk/lawsuit
	innobackupex --defaults-file=lawsuit1/backup-my.cnf --apply-log /disk/lawsuit1
	```
	
## 3. 利用docker获取表结构数据

1. docker安装mysql

	```shell
	# 查找源信息
	docker search mysql
	# 下载镜像
	docker pull mysql:5.7
	# 查看结果
	docker images | grep mysql
	```
2. 启动docker挂在gongshang数据镜像并进入镜像

	```shell
	# 启动镜像
	docker run --name mysql -p 3307:3306 -v /disk/gongshang1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=yczcjk -d mysql:5.7
	# 查看镜像ID
	docker ps
	# 进入镜像bash
	docker exec -it ${dockerID} /bin/bash
	# 进入mysql
	mysql -uroot
	# 创建外部可用的账号
	grant all privileges on prism1.* to 'test'@'%' identified by 'test123456'
	# 退出mysql
	exit
	# 更新mysql5.6 到mysql5.7
	mysql_upgrade
	# 退出镜像
	exit
	# 开启3306端口
	firewall-cmd --add-port=3307/tcp --permernent
	firewall-cmd --reload
	```
3. 接下来就可以使用navicat等工具连接刚创建的数据库，并导出表结构
4. lawsuit1数据的表结构数据导出类似，若遇到输入密码的情况，则进行如下操作

	```shell
	# 更新源
	apt-get update
	# 安装vim
	apt-get install vim
	# 编辑配置文件，在末尾加skip-grant-tables
	vim /etc/mysql/mysql.conf.d/mysqld.conf
	# 保存退出docker镜像，并重启镜像
	docker restart mysql
	# 更新mysql5.6到5.7
	mysql_upgrade
	# 进入mysql做创建账号操作，后去掉skip-grant-tables，并重启mysql镜像
	```
	
## 4. 恢复数据

1. 创建数据库

	```shell
	# 创建数据库
	create database prism1
	```
2. 导入之前获取到的表结构

3. mysql中执行如下：

	```mysql
	ALTER TABLE prism1.annual_report DISCARD TABLESPACE;
    ALTER TABLE prism1.company_abnormal_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_category_20170411 DISCARD TABLESPACE;
    ALTER TABLE prism1.company_category_code_20170411 DISCARD TABLESPACE;
    ALTER TABLE prism1.company_change_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_check_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_equity_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company DISCARD TABLESPACE;
    ALTER TABLE prism1.company_illegal_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_investor_entpub DISCARD TABLESPACE;
    ALTER TABLE prism1.company_investor DISCARD TABLESPACE;
    ALTER TABLE prism1.company_ipr_pledge_change_info_entpub DISCARD TABLESPACE;
    ALTER TABLE prism1.company_ipr_pledge_reg_info_entpub DISCARD TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_frozen_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_frozen_invalidation_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_frozen_keep_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_frozen_rem_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_judicial_shareholder_change_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_license_entpub DISCARD TABLESPACE;
    ALTER TABLE prism1.company_license DISCARD TABLESPACE;
    ALTER TABLE prism1.company_license_info_creditchina DISCARD TABLESPACE;
    ALTER TABLE prism1.company_liquidating_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_mortgage_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_other_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_punishment_info_creditchina DISCARD TABLESPACE;
    ALTER TABLE prism1.company_punishment_info DISCARD TABLESPACE;
    ALTER TABLE prism1.company_staff DISCARD TABLESPACE;
    ALTER TABLE prism1.human DISCARD TABLESPACE;
    ALTER TABLE prism1.mortgage_change_info DISCARD TABLESPACE;
    ALTER TABLE prism1.mortgage_pawn_info DISCARD TABLESPACE;
    ALTER TABLE prism1.mortgage_people_info DISCARD TABLESPACE;
    ALTER TABLE prism1.report_change_record DISCARD TABLESPACE;
    ALTER TABLE prism1.report_equity_change_info DISCARD TABLESPACE;
    ALTER TABLE prism1.report_outbound_investment DISCARD TABLESPACE;
    ALTER TABLE prism1.report_out_guarantee_info DISCARD TABLESPACE;
    ALTER TABLE prism1.report_shareholder DISCARD TABLESPACE;
    ALTER TABLE prism1.report_social_security_info DISCARD TABLESPACE;
    ALTER TABLE prism1.report_webinfo DISCARD TABLESPACE;

    ALTER TABLE prism1.court_notices DISCARD TABLESPACE;
    ALTER TABLE prism1.company_lawsuit DISCARD TABLESPACE;
    ALTER TABLE prism1.company_lawsuit_parsed_info DISCARD TABLESPACE;
    ALTER TABLE prism1.court_announcement DISCARD TABLESPACE;
	```
	
4. 将gongshang和lawsuit中prism1中的.ibd及.exp文件复制到当前输入库的prism1库中
5. 执行如下命令进行恢复

	```mysql
	ALTER TABLE prism1.annual_report IMPORT TABLESPACE;
    ALTER TABLE prism1.company_abnormal_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_category_20170411 IMPORT TABLESPACE;
    ALTER TABLE prism1.company_category_code_20170411 IMPORT TABLESPACE;
    ALTER TABLE prism1.company_change_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_check_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_equity_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company IMPORT TABLESPACE;
    ALTER TABLE prism1.company_illegal_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_investor_entpub IMPORT TABLESPACE;
    ALTER TABLE prism1.company_investor IMPORT TABLESPACE;
    ALTER TABLE prism1.company_ipr_pledge_change_info_entpub IMPORT TABLESPACE;
    ALTER TABLE prism1.company_ipr_pledge_reg_info_entpub IMPORT TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_frozen_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_frozen_invalidation_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_frozen_keep_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_frozen_rem_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_judicial_assistance_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_judicial_shareholder_change_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_license_entpub IMPORT TABLESPACE;
    ALTER TABLE prism1.company_license IMPORT TABLESPACE;
    ALTER TABLE prism1.company_license_info_creditchina IMPORT TABLESPACE;
    ALTER TABLE prism1.company_liquidating_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_mortgage_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_other_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_punishment_info_creditchina IMPORT TABLESPACE;
    ALTER TABLE prism1.company_punishment_info IMPORT TABLESPACE;
    ALTER TABLE prism1.company_staff IMPORT TABLESPACE;
    ALTER TABLE prism1.human IMPORT TABLESPACE;
    ALTER TABLE prism1.mortgage_change_info IMPORT TABLESPACE;
    ALTER TABLE prism1.mortgage_pawn_info IMPORT TABLESPACE;
    ALTER TABLE prism1.mortgage_people_info IMPORT TABLESPACE;
    ALTER TABLE prism1.report_change_record IMPORT TABLESPACE;
    ALTER TABLE prism1.report_equity_change_info IMPORT TABLESPACE;
    ALTER TABLE prism1.report_outbound_investment IMPORT TABLESPACE;
    ALTER TABLE prism1.report_out_guarantee_info IMPORT TABLESPACE;
    ALTER TABLE prism1.report_shareholder IMPORT TABLESPACE;
    ALTER TABLE prism1.report_social_security_info IMPORT TABLESPACE;
    ALTER TABLE prism1.report_webinfo IMPORT TABLESPACE;

    ALTER TABLE prism1.court_notices IMPORT TABLESPACE;
    ALTER TABLE prism1.company_lawsuit IMPORT TABLESPACE;
    ALTER TABLE prism1.company_lawsuit_parsed_info IMPORT TABLESPACE;
    ALTER TABLE prism1.court_announcement IMPORT TABLESPACE;
	```
	
6. 至此，全部恢复完成