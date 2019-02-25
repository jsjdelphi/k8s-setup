单服务器上安装kubernetes1.13.0
一、实践环境准备
1、服务器信息：
OS：CentOS 7
IP: 192.168.0.200
Hostname: 13-master
2、环境初始化
[root@13-master ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.0.200   13-master
3、关闭SWAP、防火墙和SELINUX
3.1 关闭SWAP
[root@13-master ~]# swapoff -a
[root@13-master ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Mon Feb 25 13:40:05 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=bd9a8c9e-dc36-47a2-b9e3-19c0ced472c8 /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0

3.2 关闭防火墙
[root@13-master ~]# setenforce 0
[root@13-master ~]# systemctl stop firewalld && systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.

3.3 关闭SELINUX
# vi /etc/selinux/config
SELINUX=disabled

4、开启forward
# iptables -P FORWARD ACCEPT

5、配置系统路由参数

# echo "
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
" >> /etc/sysctl.conf

# sysctl -p
6、加载ipvs相关内核模块
# modprobe ip_vs
# modprobe ip_vs_rr
# modprobe ip_vs_wrr
# modprobe ip_vs_sh
# modprobe nf_conntrack_ipv4
# lsmod | grep ip_vs

7、配置国内Kubernetes源地址
# vi /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

二、安装Docker
# yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-18.06.1.ce-3.el7.x86_64.rpm
# systemctl enable docker && systemctl restart docker
# docker --version

[root@13-master ~]# docker --version
Docker version 18.06.1-ce, build e68fc7a

三、安装kubeadm、kubelet和kubectl
1、工具说明
• kubeadm: 部署集群用的命令
• kubelet: 在集群中每台机器上都要运行的组件，负责管理pod、容器的生命周期
• kubectl: 集群管理工具
2、yum安装
# yum install -y kubelet-1.13.0 kubeadm-1.13.0 kubectl-1.13.0 ipvsadm
3、 配置kubelet使用国内pause镜像，修改cgroup配置
# vi /etc/sysconfig/kubelet
添加配置如下：
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
4、启动kubelet
# systemctl daemon-reload
# systemctl enable kubelet && systemctl start kubelet

如下图:
[root@13-master ~]# vi /etc/sysconfig/kubelet 
[root@13-master ~]# cat /etc/sysconfig/kubelet 
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
[root@13-master ~]# systemctl daemon-reload
[root@13-master ~]# systemctl enable kubelet && systemctl start kubelet
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.

四、下载镜像
1、查看依赖需要安装的镜像列表
# kubeadm config images list       

[root@13-master k8s-install]# kubeadm config images list 
k8s.gcr.io/kube-apiserver:v1.13.3
k8s.gcr.io/kube-controller-manager:v1.13.3
k8s.gcr.io/kube-scheduler:v1.13.3
k8s.gcr.io/kube-proxy:v1.13.3
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.2.24
k8s.gcr.io/coredns:1.2.6

2、生成kubeadm.conf文件
[root@13-master k8s-install]# cat kubeadm.conf 
apiVersion: kubeadm.k8s.io/v1beta1
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: 13-master
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: ""
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/k8s-setup
kind: ClusterConfiguration
kubernetesVersion: v1.13.0
networking:
  dnsDomain: cluster.local
  podSubnet: ""
  serviceSubnet: 10.244.0.0/16
scheduler: {}

3、下载需要用到的镜像
# kubeadm config images pull --config kubeadm.conf
[root@13-master k8s-install]# kubeadm config images pull --config kubeadm.conf 
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-apiserver:v1.13.0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-controller-manager:v1.13.0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-scheduler:v1.13.0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-proxy:v1.13.0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/k8s-setup/pause:3.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/k8s-setup/etcd:3.2.24
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/k8s-setup/coredns:1.2.6

自动下载v1.13需要用到的镜像，执行 docker images 可以看到下载好的镜像列表：
[root@13-master k8s-install]# docker images
REPOSITORY                                                            TAG                 IMAGE ID            CREATED             SIZE
registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-proxy                v1.13.0             8fa56d18961f        2 months ago        80.2MB
registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-scheduler            v1.13.0             9508b7d8008d        2 months ago        79.6MB
registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-apiserver            v1.13.0             f1ff9b7e3d6e        2 months ago        181MB
registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-controller-manager   v1.13.0             d82530ead066        2 months ago        146MB
registry.cn-hangzhou.aliyuncs.com/k8s-setup/coredns                   1.2.6               f59dcacceff4        3 months ago        40MB
registry.cn-hangzhou.aliyuncs.com/k8s-setup/etcd                      3.2.24              3cab8e1b9802        5 months ago        220MB
registry.cn-hangzhou.aliyuncs.com/k8s-setup/pause                     3.1                 da86e6ba6ca1        14 months ago       742kB

4、docker tag 镜像
镜像下载好后，我们还需要tag下载好的镜像，让下载好的镜像都是带有k8s.gcr.io标识的，如果不打tag变成k8s.gcr.io，那么后面用kubeadm安装会出现问题，因为kubeadm里面只认google自身的模式。
我们执行下面命令即可完成tag标识更换：
docker tag registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-apiserver:v1.13.0 k8s.gcr.io/kube-apiserver:v1.13.0
docker tag registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-controller-manager:v1.13.0 k8s.gcr.io/kube-controller-manager:v1.13.0
docker tag registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-scheduler:v1.13.0 k8s.gcr.io/kube-scheduler:v1.13.0
docker tag registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-proxy:v1.13.0 k8s.gcr.io/kube-proxy:v1.13.0
docker tag registry.cn-hangzhou.aliyuncs.com/k8s-setup/pause:3.1 k8s.gcr.io/pause:3.1
docker tag registry.cn-hangzhou.aliyuncs.com/k8s-setup/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24
docker tag registry.cn-hangzhou.aliyuncs.com/k8s-setup/coredns:1.2.6 k8s.gcr.io/coredns:1.2.6

5、docker rmi 清理下载的镜像
执行完上面tag镜像的命令，我们还需要把带有 registry.aliyuncs.com 标识的镜像删除，执行：
docker rmi registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-apiserver:v1.13.0
docker rmi registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-controller-manager:v1.13.0
docker rmi registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-scheduler:v1.13.0
docker rmi registry.cn-hangzhou.aliyuncs.com/k8s-setup/kube-proxy:v1.13.0
docker rmi registry.cn-hangzhou.aliyuncs.com/k8s-setup/pause:3.1
docker rmi registry.cn-hangzhou.aliyuncs.com/k8s-setup/etcd:3.2.24
docker rmi registry.cn-hangzhou.aliyuncs.com/k8s-setup/coredns:1.2.6

6、查看下载的镜像列表
执行docker images命令，即可查看到，这里结果如下，您下载处理后，结果需要跟这里的一致：
[root@13-master k8s-install]# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.13.0             8fa56d18961f        2 months ago        80.2MB
k8s.gcr.io/kube-apiserver            v1.13.0             f1ff9b7e3d6e        2 months ago        181MB
k8s.gcr.io/kube-controller-manager   v1.13.0             d82530ead066        2 months ago        146MB
k8s.gcr.io/kube-scheduler            v1.13.0             9508b7d8008d        2 months ago        79.6MB
k8s.gcr.io/coredns                   1.2.6               f59dcacceff4        3 months ago        40MB
k8s.gcr.io/etcd                      3.2.24              3cab8e1b9802        5 months ago        220MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        14 months ago       742kB

五、部署master节点
1、kubeadm init 初始化master节点
# kubeadm init --kubernetes-version=v1.13.0 --pod-network-cidr=10.244.0.0/16
2、初始化成功后，/etc/kubernetes/会生成下面文件
[root@13-master ~]# cd /etc/kubernetes/
[root@13-master kubernetes]# ll
total 36
-rw-------. 1 root root 5449 Feb 25 21:05 admin.conf
-rw-------. 1 root root 5489 Feb 25 21:05 controller-manager.conf
-rw-------. 1 root root 5477 Feb 25 21:05 kubelet.conf
drwxr-xr-x. 2 root root  113 Feb 25 21:05 manifests
drwxr-xr-x. 3 root root 4096 Feb 25 21:05 pki
-rw-------. 1 root root 5437 Feb 25 21:05 scheduler.conf
3、同时最后会生成一句话
kubeadm join 192.168.0.200:6443 --token 87pp2b.0pnrld0brjidgjty --discovery-token-ca-cert-hash sha256:c50c1bd655a322096a5414a841b8992ab304e6260e6e0a1a3b93f7877f5f2901

注意：这个我们记录下，到时候添加node的时候要用到。

4、配置环境变量
# vi ~/.bash_profile
加入一行：export KUBECONFIG=/etc/kubernetes/admin.conf
# source ~/.bash_profile

5、flannel安装
使用kube-flannel.yml文件
# wget  https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
# kubectl apply -f kube-flannel.yml

6、安装验证
# kubectl get pod -n kube-system
# kubectl get node

[root@13-master ~]# kubectl get pod -n kube-system
NAME                                READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-g7pdk            1/1     Running   0          61m
coredns-86c58d9df4-hjwvw            1/1     Running   0          61m
etcd-13-master                      1/1     Running   0          61m
kube-apiserver-13-master            1/1     Running   0          61m
kube-controller-manager-13-master   1/1     Running   0          61m
kube-flannel-ds-amd64-flllq         1/1     Running   0          58m
kube-proxy-7gv85                    1/1     Running   0          61m
kube-scheduler-13-master            1/1     Running   0          61m
[root@13-master ~]# kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
13-master   Ready    master   62m   v1.13.0

注意：
设置master可以分配pod
master节点上一般不会再分配pod，以保障master的高效运行， 如果仅有1台服务器，没有node的情况下，需要执行如下语句使master可以分配pod
# kubectl taint nodes --all node-role.kubernetes.io/master-
