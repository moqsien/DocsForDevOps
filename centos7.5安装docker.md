# CentOS 7.5 安装docker

1. 更新本地
	```shell
	yum update
	```

2. 查看本地源中是否有docker

	```shell
	yum list docker-ce --showduplicates | sort -r
	```

3. 添加docker源

	```shell
	yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
	```

4. 再次检查是否有docker

	```shell
	yum list docker-ce --showduplicates | sort -r
	```

5. 安装docker-ce

	```shell
	yum install docker-ce
	```

6. 增加docker用户组，有可能已经存在该用户组，导致添加失败，添加之前可以先检查下是否有该用户组

	```shell
	groupadd docker
	```

7. 将需要执行docker的普通账号加入到docker用户组

	```shell
	usermod -a -G docker work
	```

8. 启动docker服务

	```shell
	service docker start
	```

~~愉快的玩耍吧~~

