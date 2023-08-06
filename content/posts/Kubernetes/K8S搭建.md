---
title: "手动搭建k8s"
date: 2023-08-06
tags: ["k8s"]
---


镜像来源于阿里云的centos镜像，安装VMware过程省略
搭建一个master节点，三个node节点。配置都是2c8g
# 安装前准备
所有的节点都需要进行此操作

1：所有节点禁止防火墙
``` bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```
2：禁用SELinux,使得容器可以访问主机文件系统
``` bash
vim /etc/sysconfig/selinux 
# 将这一行修改为disabled
SELINUX=disabled
```
3：关闭swap
``` bash
swapoff -a
vim /etc/fstab
# 删除swap相关行 /mnt/swap swap swap defaults 0 0 这一行或者注释掉这一行
vim /etc/sysctl.conf
# 仅在内存不足的情况下–当剩余空闲内存低于vm.min_free_kbytes limit时，使用交换空间
vm.swappiness = 0 
# 使配置生效
sysctl -p
```
4：配置k8s内核
``` bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
5：开启cggroup
先查询是否开启
``` bash
cat /boot/config-`uname -r` | grep CGROUP
```
然后升级内核版本（4以上不用升级）
``` bash
# 更新yum
yum -y update
# 导入ELRepo仓库的公共密钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# 安装ELRepo仓库的yum源
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# 查看系统可用的内核包
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
# 安装最新版本内核 --enablerepo 选项开启 CentOS 系统上的指定仓库。默认开启的是 elrepo，这里用 elrepo-kernel 替换。
yum --enablerepo=elrepo-kernel install kernel-ml
# 查看所有可用的内核
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
# 开始升级内核的默认版本
vim /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=0
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=cl/root rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
# 生成配置文件并重启
grub2-mkconfig -o /boot/grub2/grub.cfg
rebort
# 验证内核是否已经更新
uname -r
```
6：进行时间的同步
``` bash
# 启动chronyd服务
systemctl start chronyd
systemctl enable chronyd
date 
``` 
7：禁止iptable和firewalld
k8s和docker会生成大量的iptables规则，为了防止和系统的规则混淆和方便测试，选择关闭系统的规则
``` bash
# 1 关闭firewalld服务
systemctl stop firewalld
systemctl disable firewalld
# 2 关闭iptables服务
systemctl stop iptables
systemctl disable iptables
```
8：禁用selinux 
不关闭这个集群操作会产生许多奇葩问题，方便测试选择关闭
``` bash
vi /etc/selinux/config
SELINUX=disabled
```
注意重启机器 

9：配置ipvs功能
k8s中的service有两种模型，一种是基于iptables的，一种是基于ipvs的
ipvs的性能相对较高，但是使用此模式需要手动加载
``` bash

# 1.安装ipset和ipvsadm
yum install ipset ipvsadm -y
# 2.添加需要加载的模块写入脚本文件
cat <<EOF> /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
# 3.为脚本添加执行权限
chmod +x /etc/sysconfig/modules/ipvs.modules
# 4.执行脚本文件
bin/bash /etc/sysconfig/modules/ipvs.modules
# 5.查看对应的模块是否加载成功
lsmod | grep -e ip_vs -e nf_conntrack_ipv4

```

10：安装docker
``` bash
# 10.1、切换镜像源
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

# 10.2、查看当前镜像源中支持的docker版本
yum list docker-ce --showduplicates

# 10.3、安装特定版本的docker-ce
# 必须制定--setopt=obsoletes=0，否则yum会自动安装更高版本
yum install --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7 -y

# 10.4、添加一个配置文件
#Docker 在默认情况下使用Vgroup Driver为cgroupfs，而Kubernetes推荐使用systemd来替代cgroupfs
mkdir /etc/docker
cat <<EOF> /etc/docker/daemon.json
{
	"exec-opts": ["native.cgroupdriver=systemd"],
	"registry-mirrors": ["https://kn0t2bca.mirror.aliyuncs.com"]
}
EOF

10.5、启动dokcer
  systemctl restart docker
  systemctl enable docker
```
11：安装k8s组件
``` bash

# 11.1、切换成国内的镜像源
# 11.2、编辑/etc/yum.repos.d/kubernetes.repo,添加下面的配置
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgchech=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
			http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

# 11.3、安装kubeadm、kubelet和kubectl
yum install --setopt=obsoletes=0 kubeadm-1.17.4-0 kubelet-1.17.4-0 kubectl-1.17.4-0 -y

# 11.4、配置kubelet的cgroup
#编辑/etc/sysconfig/kubelet, 添加下面的配置
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"

# 11.5、设置kubelet开机自启
systemctl enable kubelet

```
12：准备集群镜像
``` bash

images=(
	kube-apiserver:v1.17.4
	kube-controller-manager:v1.17.4
	kube-scheduler:v1.17.4
	kube-proxy:v1.17.4
	pause:3.1
	etcd:3.4.3-0
	coredns:1.6.5
)

for imageName in ${images[@]};do
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
	docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName 
done

```
13：集群初始化
下列命令只需要在master节点进行操作
``` bash

# 创建集群
kubeadm init \
	--apiserver-advertise-address=192.168.90.109 \
	--image-repository registry.aliyuncs.com/google_containers \
	--kubernetes-version=v1.17.4 \
	--service-cidr=10.96.0.0/12 \
	--pod-network-cidr=10.244.0.0/16
# 创建必要文件
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
此命令执行完毕后会生成token，需要根据这个token去节点执行join
``` bash
kubeadm join 192.168.0.109:6443 --token awk15p.t6bamck54w69u4s8 \
    --discovery-token-ca-cert-hash sha256:a94fa09562466d32d29523ab6cff122186f1127599fa4dcd5fa0152694f17117 
```

然后再master上查看节点是否已经加入
``` bash
kubectl get nodes
```
出现此状态就算成功
``` bash
NAME    STATUS   ROLES     AGE   VERSION
master  NotReady  master   6m    v1.17.4
node1   NotReady   <none>  22s   v1.17.4
node2   NotReady   <none>  19s   v1.17.4
node3   NotReady   <none>  19s   v1.17.4
```
接下来安装网络插件 只需要在master节点安装
此过程如果遇见问题，使用 journalctl -f -u kubelet.service 查看原因
``` bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

mkdir -p /etc/cni/net.d 
vim /etc/cni/net.d/10-flannel.conf
{
 "name":"cbr0",
 "cniVersion":"0.3.1",
 "type":"flannel",
 "deledate":{
    "hairpinMode":true,
    "isDefaultGateway":true
  }

}

# 其余的node需要重置
kubeadm reset
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
##重启kubelet
systemctl restart kubelet
##重启docker
systemctl restart docker
# 切回来master 执行
kubectl apply -f kube-flannel.yml
```
过个一两分钟就可以看到node处于reday状态了
![](https://lbnote-1304107363.cos.ap-nanjing.myqcloud.com/20221111161421.png)
此刻就可以进行部署测试了

参考：
https://gitee.com/yooome/golang/blob/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B-%E8%B0%83%E6%95%B4%E7%89%88/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B.md#261-%E6%A3%80%E6%9F%A5%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9A%84%E7%89%88%E6%9C%AC

https://blog.csdn.net/yingdehang/article/details/118693039

https://blog.csdn.net/qq_21277357/article/details/110120050
