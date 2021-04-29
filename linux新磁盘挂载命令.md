# linux新磁盘挂载命令

找到要挂在的新磁盘，如找到的新磁盘为/dev/sdd
常用命令有如下两个：`fdisk -l`和`df -h`

1. 格式化磁盘：

```shell
mkfs -t xfs /dev/sdd
```

2. 新建待挂载文件夹

```shell
mkdir kafka
```

3. 挂载硬盘

```shell
mount /dev/sdd kafka
```

4. 加入开机启动

```shell
blkid /dev/sdd
vim /etc/fstab
```