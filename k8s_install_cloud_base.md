- [基于混合云的集群搭建](#基于混合云的集群搭建)
  - [环境](#环境)
    - [云厂商](#云厂商)
    - [机器（2台）](#机器2台)
      - [系统](#系统)
      - [硬件配置（系统盘size看个人需求）](#硬件配置系统盘size看个人需求)
  - [部署](#部署)
    - [角色设定](#角色设定)
    - [主机名设定](#主机名设定)
    - [系统初始化配置（所有机器都要配置）](#系统初始化配置所有机器都要配置)
    - [安装Docker（所有机器都要配置）](#安装docker所有机器都要配置)
    - [配置k8s的yum源地址](#配置k8s的yum源地址)
    - [安装kubernetes](#安装kubernetes)
    - [安装flannel网络插件](#安装flannel网络插件)
    - [检查集群状态](#检查集群状态)
  - [出现的问题](#出现的问题)
    - [kubeadm init 失败](#kubeadm-init-失败)
# 基于混合云的集群搭建
## 环境
### 云厂商
* 阿里云 + 腾讯云
### 机器（2台）
#### 系统
* CentOS8.x-86_x64
#### 硬件配置（系统盘size看个人需求）
* 阿里云 CPU2核，内存4GB，硬盘 80GB
* 腾讯云 CPU2核，内存4GB，硬盘 60GB

## 部署
### 角色设定
* 一个主节点master一个工作节点node
* 根据服务器性能，较好的当做 worker 节点，较差的当做 master 节点
### 主机名设定
* 两台设备主机名分别为 master node
```
hostnamectl set-hostname master
hostnamectl set-hostname node
```
* 修改master的hosts文件,将下面公共IP内容，替换成云服务器公IP
```
cat >> /etc/hosts << EOF
[公IP] master
[公IP] node
EOF
```
### 系统初始化配置（所有机器都要配置）
* 关闭防火墙
```
systemctl stop firewalld  # 临时
systemctl disable firewalld # 永久
```
* 关闭seLinux
```
sed -i 's/enforcing/disabled/' /etc/selinux/config # 永久 
setenforce 0 # 临时
```
* 关闭swap
```
swapoff -a # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab # 永久
```
* 将桥接的 IPv4 流量传递到 iptables 的链
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
 
cat > /etc/sysctl.d/k8s.conf << EOF 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system 
```
### 安装Docker（所有机器都要配置）
* 以下操作在所有机器进行
* k8s-1.18版本之前 + centos-7.x 版本
```
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce-18.06.1.ce-3.el7
systemctl enable docker && systemctl start docker
docker --version
```
* k8s-1.18版本之后 + centos-8.x 版本
```
yum install https://download.docker.com/linux/fedora/30/x86_64/stable/Packages/containerd.io-1.2.6-3.3.fc30.x86_64.rpm
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce 

systemctl enable docker && systemctl start docker
docker --version
```
* 创建/etc/docker/daemon.json配置文件并指定镜像仓库
```
cat > /etc/docker/daemon.json << EOF 
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```
* 重启docker
```
systemctl restart docker
```
### 配置k8s的yum源地址
* 为了以后下载软件包能够快一点，这里使用阿里的yum源
```
vim /etc/yum.repos.d/kubernetes.repo
```
添加下面的配置内容
```
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```
### 安装kubernetes
* 所有机器安装 kubelet，kubeadm，kubectl
* 所有机器设置开机启动kubelet
```
yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9
systemctl enable kubelet
```
* master上部署k8s
```
kubeadm init \
--image-repository=registry.aliyuncs.com/google_containers \ 
--pod-network-cidr=10.244.0.0/16 \ 
--service-cidr=10.96.0.0/12 \ 
--kubernetes-version=v1.20.9 \ 
--apiserver-advertise-address=[master public ip]
```
若执行命令后此时会卡住，出现下面情况
```
Waiting for the kubelet to boot up the control plane as static Pods from directory “/etc/kubernetes/manifests”. This can take up to 4m0s
Initial timeout of 40s passed
```
由于etcd配置文件不正确，所以etcd无法启动，要对该文件进行修改。
文件路径"/etc/kubernetes/manifests/etcd.yaml"。
打开文件后要修改以下几个点
1. 找到 –listen-client-urls 把"–listen-client-urls"后面的公网ip删除
2. 找到 –listen-peer-urls 把"–listen-peer-urls"改为本地的地址。

稍后master节点初始化就会完成，初始化成功后显示如下
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join xxx.xxx.xxx.xxx:6443 --token xxxx.wkiew6dgduusu6ie \
    --discovery-token-ca-cert-hash sha256:xxxxx883b33770c50455f1f19306f771dab67b28f9a986fxxxxx
```
* 在从worker节点上执行上面log结果中的 kubeadm join .... 命令,将从节点加入集群
```
kubeadm join xxx.xxx.xxx.xxx:6443 --token xxxx.wkiew6dgduusu6ie \
    --discovery-token-ca-cert-hash sha256:xxxxx883b33770c50455f1f19306f771dab67b28f9a986fxxxxx
```
### 安装flannel网络插件
* 在master上执行下面脚本
```
kubectl apply -f https://gitee.com/www.freeclub.com/blog-images/raw/master/source/kube-flannel.yml
```
结果如下
```
podsecuritypolicy.policy/psp.flannel.unprivileged created
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
clusterrole.rbac.authorization.k8s.io/flannel created
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
```
### 检查集群状态
* 在master节点上执行
```
kubectl get node
```
结果如下
```
[root@master ~]# kubectl get node
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   45h   v1.20.9
node     Ready    <none>                 45h   v1.20.9
[root@master ~]# 
```
```
kubectl get pods -n kube-system
```
结果如下
```
NAME                             READY   STATUS    RESTARTS   AGE
coredns-7f89b7bc75-52ngr         1/1     Running   0          45h
coredns-7f89b7bc75-qtfds         1/1     Running   0          45h
etcd-master                      1/1     Running   0          45h
kube-apiserver-master            1/1     Running   2          45h
kube-controller-manager-master   1/1     Running   0          45h
kube-flannel-ds-amd64-flqw5      1/1     Running   0          45h
kube-flannel-ds-amd64-qhq82      1/1     Running   0          45h
kube-proxy-f2gp8                 1/1     Running   0          45h
kube-proxy-ns56s                 1/1     Running   0          45h
kube-scheduler-master            1/1     Running   0          45h
[root@master ~]# 
```
## 出现的问题
### kubeadm init 失败
* 6443端口是否开放，是否被监听
* 查询kubelet服务统计信息，查看log日志
