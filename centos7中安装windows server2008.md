# Centos7中安装Windows server 2008

**所需要的**

- kvm
- windows server 2008镜像

## 安装过程

### 检查系统是否支持

1. cat /proc/cpuinfo | egrep 'vmx|svm'

### 安装KVM

1. 安装KVM基础包：yum -y install kvm
2. 安装KVM管理工具：yum -y install qemu-kvm python-virtinst libvirt libvirt-python virt-manager libguestfs-tools bridge-utils virt-install

3. lsmod | grep kvm
4. 启动服务：systemctl start libvirtd.service
5. 查看状态：systemctl status libvirtd.service

### 安装windows server 2008

1. 安装：virt-install --connect qemu:///system --name windows --memory=16384--vcpus=8 --disk path=/disk/vms/windows.qcow2,device=disk,format=qcow2,bus=ide,cache=none,size=50 --cdrom /disk/vms/cn_windows_server_2008_r2_standard_enterprise_datacenter_web_x64_dvd_x15-50360.iso --os-type=windows --network bridge=virbr0,model=virtio,model=e1000 --hvm --virt-type=kvm --noautoconsole --graphics vnc,listen=0.0.0.0,port=5902  
2. 开启防火墙： firewall-cmd --add-ports=5902/tcp --permernant
3. 重启防火墙：firewall-cmd --reload
4. 使用vnc-viewer连接该节点完成安装