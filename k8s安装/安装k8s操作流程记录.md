[toc]

# 安装k8s操作流程记录

参考文件地址
[1集群](https://blog.51cto.com/3241766/2405624)
[15主备](https://blog.51cto.com/3241766/2463125)
[16高可用](https://blog.51cto.com/3241766/2467865)

**环境说明：**

| 主机名   | 操作系统版本 | IP              | 备注       |
| -------- | ------------ | --------------- | ---------- |
| master01 | CentOS7.9    | 192.168.205.133 | Master主机 |
|work01|CentOS7.9|192.168.205.134|Node主机|
|work02|CentOS7.9|192.168.205.135|Node主机|

**安装软件说明：**

|软件名称|版本号|安装主机|备注|
|---|---|---|---|
|docker|19.03.9|Master+Node|容器|
|kubelet|1.19.4|Master+Node|主要服务|
|kubeadm|1.19.4|Master|启动集群及加入集群|
|kubectl|1.19.4|Master|命令行|
|kubernetes-dashboard|2.0.4|Master|UI操作面板|




## 1 docker环境准备工作

### 1.1 安装依赖包

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```

### 1.2 设置docker源

```shell
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### 1.3 docker安装版本查看

```shell
yum list docker-ce --showduplicates|sort -r
```

### 1.4 安装docker

```shell
yum install docker-ce
```

### 1.5 镜像加速

登录[地址](https://cr.console.aliyun.com); 找到镜像中心->镜像加速器，查看相关配置

```shell
1. mkdir -p /etc/docker
2. tee /etc/docker/daemon.json <<-'EOF'
{
	"registry-mirrors": ["https://maxlibsu.mirror.aliyuncs.com"]
}
EOF
3. systemctl daemon-reload
4. systemctl restart docker
```

## 2 安装K8S具体操作

### 2.1 修改hostname文件并使立即生效

```shell
hostnamectl set-hostname master01
sysctl kernel.hostname=$(cat /etc/hostname)
```

### 2.2 修改hosts文件

```shell
cat >> /etc/hosts << EOF
192.168.205.133 master01
192.168.205.134 work01
192.168.205.135 work02
EOF
```

### 2.3 验证mac地址和UUID是否唯一

```shell
cat /sys/class/net/ens33/address
cat /sys/class/dmi/id/product_uuid 
```

### 2.4 内核参数修改

```shell
cat << EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
```

### 2.5 设置kubernetes源

```shell
cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum clean all
yum -y makecache
```
> [] 中括号中的是repository id，唯一，用来标识不同仓库
> name 仓库名称，自定义
> baseurl 仓库地址
> enabled 是否启用该仓库，默认为1表示启用
> gpgcheck 是否验证从该仓库获得程序包的合法性，1为验证
> repo_gpgcheck 是否验证元数据的合法性，元数据就是程序包列表，1为验证
> gpgkey 数字签名的公钥文件所在位置，如果gpgcheck值为1，此处就需要指定gpgkey文件的位置，如果gpgcheck值为0就不需要此项了
> 

### 2.6 Master节点安装

```shell
yum install -y kubelet kubeadm kubectl

systemctl enable kubelet
systemctl start kubelet
```

> kubelet 运行在集群所有节点上，用于启动Pod和容器等对象的工具
> kubeadm 用于初始化集群，启动集群的命令工具
> kubectl 用于和集群通信的命令行，通过kubectl可以部署和管理应用，查看各种资源，创建、删除和更新各种组件
> 

### 2.7 下载镜像

```shell
镜像下载的脚本image.sh

#!/bin/bash
url=registry.cn-hangzhou.aliyuncs.com/google_containers
version=v1.19.4
images=(`kubeadm config images list --kubernetes-version=$version|awk -F '/' '{print $2}'`)
for imagename in ${images[@]}; do
  docker pull $url/$imagename
  docker tag $url/$imagename k8s.gcr.io/$imagename
  docker rmi -f $url/$imagename
done
```

------

下载镜像

```shell
chmod u+x image.sh
./image.sh
docker images
```

### 2.8 初始化Master

```shell
kubeadm init --apiserver-advertise-address 192.168.205.133 --pod-network-cidr=10.244.0.0/16
# 得到类似如下信息，复制出来
kubeadm join 192.168.205.133:6443 --token wqczze.m4hqc5z00ljq9v4e \
    --discovery-token-ca-cert-hash sha256:7dc10399821a534200c8a898818f7b960adc9ba8b228907abea37546d4542078
```

> --apiserver-advertise-address: 指定Master的`interface`
> --pod-network-cidr: 指定pod网络的范围，这里使用`flannel`网络方案

### 2.9 加载环境变量

```shell
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source .bash_profile
```

### 2.10 安装pod网络

[下载地址](https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml)
下载保存为kube-flannel.yml
使用`kubectl apply -f kube-flannel.yml`使生效

### 2.11 删除Master污点标记

```shell
# 查看污点
kubectl describe node master01|grep -i taints
# 删除污点
kubectl taint nodes master01 node-role.kubernetes.io/master-
```

### 2.12 Node节点安装

前两步同Master节点前两部安装kubelet kubeadm kubectl并配置镜像
使用初始化Master中复制出来的信息，将Node节点加入集群，若上面token过期，Master节点上执行如下操作

```shell
# 查看是否过期
kubeadm token list
# 生成新的token令牌
kubeadm token create
# 生成新的加密串
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null|\
openssl dgst -sha256 -hex | sed 's/^.*//'
```

### 2.13 防火墙

开启端口有：2379(etcd客户端监听),2380(etcd节点间内部通讯),6443(kubernetes api端口),8443(LB VIP监听的服务),10250(kubelet服务监听),10251(controller manager),10252(scheduler),30000-32767/tcp（外部暴露）,8472/udp（dashboard依赖端口，flannel通讯）
开启允许IP伪装：`firewall-cmd --add-masquerade --permernent`，重新加载使生效`firewall-cmd --reload`

### 2.14 安装Dashboard

[下载地址](https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml)
命名为`kubernetes-dashboard.yaml`修改相关配置，并配置管理集群账号

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort # 类型为NodePort，为下面暴露端口准备
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001 # 此处为增加暴露端口
  selector:
    k8s-app: kubernetes-dashboard
    
# ---------------------下面为新增内容----------------------------
---

# --------------- dashboard-admin -------------- #
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```
使生效`kubectl apply -f kubernetes-dashboard.yaml`

查看相关状态

```shell
kubectl get deployment kubernetes-dashboard -n kubernetes-dashboard
kubectl get pods -n kubernetes-dashboard -o wide
kubectl get services -n kubernetes-dashboard
```

查看令牌

```shell
kubectl describe secrets -n kubernetes-dashboard dashboard-admin
```

使用浏览器[访问](https://192.168.205.133:30001)，输入上一步获取的令牌进入查看

### 2.15 测试集群

**命令方式**

```shell
kubectl run httpd-app --image=httpd --replicas=3
```

**配置方式**
新建nginx.yaml并填入如下内容

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
spec:
  type: NodePort
  ports:
  - name: nginx
    nodePort: 30002
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
```

部署上述服务`kubectl apply -f nginx.yaml`

### 2.16 查看状态命令

```shell
# 查看节点状态
kubectl get nodes
# 查看pod状态
kubectl get pod --all-namespaces
# 查看副本数
kubectl get deployments
kubectl get pod -o wide
# 查看deployment详细信息
kubectl describe deployments
# 查看集群状态
kubectl get cs
```

在查看集群状态时发现`scheduler`和`controller-manager`状态错误，修改`/etc/kubernetes/manifests`路径下的`kube-scheduler.yaml`和`kube-controller-manager.yaml`找到`- --port=0`并注释该行`# - --port=0`，后重启kubelet即可`systemctl restart kubelet`

### 2.17 命令补全

安装`bash-completion`

```shell
# 安装
yum install -y bash-completion
# 加载
source /etc/profile.d/bash_completion.sh
# kubelet命令补全
echo "source <(kubectl completion bash)" >> ~/.bash_profile
source .bash_profile
```