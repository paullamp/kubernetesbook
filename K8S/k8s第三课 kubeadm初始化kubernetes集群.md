## k8s第三课 kubadmin初始化kubernetes集群

### k8s集群组件及通信结构
![](./k8s-struct.png)

###k8s deploy struct
k8s集群支持两种部署模式，一种为传统的使用二进程程序或yum安装，一种基于自管理的pod安装。
- 直接使用二进制，yum等安装工具进行部署,即传统的部署模式。需要手动安装，修改配置文件，并且部署证书。将相关组件部署为系统级的守护进程。
- 基于pod进行部署主要的服务，现在运行的是static pod ,自管理的pod,而非控制器管理的pod.
![](./k8s-deploy-struct.png)

### 基于kubeadm进行安装
- 所有节点都必须安装kubeadm, docker, kubelet
- master节点使用kubadmin init进行初始化。安装过程是将k8s自己的组件(apiserver/scheduler/controller-manager/etcd)都通过pod方式运行和管理
- node节点上，都运行了kube-proxy的pod
- kubeadm有强大的功能，可以尝试其他功能。
	- 直接使用kubeadm进行多master集群配置
	- 使用自定义证书，不再使用安装过程中自动生成。
	- 使用自定义的registry

### kubeadmin
####安装流程:
0. 确保iptables , firewall 服务没有开启　systemctl disable firewalld.service
1. master, nodes : 安装kubelet, kubeadm, docker,kubectl(master上安装，其他节点可选)
2. master: kubeadm init
	- https://github.com/kubernetes/kubeadm/blob/master/docs/design/design_v1.10.md 
	- 预检查
	- 分析kubeadm的安装过程以及原理
	- 获取镜像资源 
3. nodes : kubeadm join xxx 

####安装详细过程:
0.基础环境准备
- 内核参数
检查内核参数,需保证为开启状态：
cat /proc/sys/net/bridge/bridge-nf-call-ip6tables
cat /proc/sys/net/bridge/bridge-nf-call-iptables
- swap
	-　确保swap为关闭状态
 
1.准备yum安装需要的软件包
- docker-ce软件repo:
cd /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

- kubernetes 软件repo
/etc/yum.repos.d/kubernetes.repo
~~~ bash
[kubernetes]
name=kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/linux
enabled=1
gpgcheck=0
~~~
- 检查repo是否有效`yum repolist`

- 为其余节点配置相同的repo
~~~ bash
scp kubernetes.repo docker-ce.repo root@node01:/etc/yum.repos.d/
scp kubernetes.repo docker-ce.repo root@node02:/etc/yum.repos.d/
rpm --import yum-key.gpg
rpm --import rpm-key.gpg
~~~


2.安装必备的软件包
- master:
	- yum -y install docker-ce kubeadm kubelet kubectl
- node: 
	- yum -y install docker-ce kubeadm kubelet

3.docker基础设置
- 所有节点的docker翻墙设置
vim /usr/lib/systemd/system/docker.service 
在[service]字段中，定义一个环境变量：
~~~bash
Enviroment="HTTPS_PROXY=http://www.ik8s.io:10080"
Enviroment="NO_PROXY=127.0.0.0/8,172.20.0.0/20"
~~~
- 应用docker配置并启动docker/kubelet服务
~~~ bash
systemctl deamon-reload
systemctl enable docker
systemctl enable kubelet
systemctl start docker
~~~

4.使用kubeadm进行master初始化
- 不需要启用swap
	- kubeadm init --kubenetes-version=v1.11 --pod-network-cidr=10.244.0.0./16 --service-cidr=10.96.0.0/12  


- 需要开启swap功能
    - 需要修改/etc/sysconfig/kubelet为KUBELET_EXTRA_ARGS="--fail-swap-on=false"
    - kubeadm init 时增加 --ignore-preflight-errors=Swap
    - kubeadm init --kubenetes-version=v1.11 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap 

- 如果需要预先加载镜像
	- kubeadm config image pull

- 使用的基础镜像说明:
|image-name|端口|功能|
|:--:|:--:|:--:|
|kube-apiserver-amd64|6443| 提供restful api服务,接收来自集群内部/外部客户端的请求|
|kube-controller-manager-amd64||controller控制器，用于监控各控制器状态|
|kube-scheduler-amd65||用于对资源进行监视调度，以让pod运行在最佳资源节点上|
|etcd-amd64|2379/2380|存储k8s集群的所有信息|
|pause|| 基础架构镜像，可以为运行在pod中的容器提供网络/存储卷等基础环境|
|CoreDns|| 集群内部的dns服务，用于为service 提供名称解析，经历了skydns->kubedns->coredns的演进|
|kube-proxy||1.11版本后，默认支持ipvs,1.10之前默认支持的是iptables|


5.kubeadm init 完成后，会有以下提示：
- 如何准备用户kubectl的配置文件
    - To start using your cluster, you need to run the following as a reguler user
    - mkdir -p $HOME/.kube
    - sudo cp -i /etc/kubenetes/admin.conf $HOME/.kube/config
    - sudo chown $(id -u):$(id -g) $HOME/.$kube/config
    - 配置文件中包含证书信息，apiserver地址信息
- 如何为kubernetes集群准备网络
    - kubectl apply -f [podnetwork].yaml
    - https://kubernetes.io/docs/concepts/cluster-administration/addons/
- 如何将其他docker-host加入到此k8s集群
    - kubeadm join aipserver-ip:apiserver-port --token xxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxx
kubectl apply -f https://
6.部署网络组件
- 部署flannel
	- `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`
	
7.kubectl 基础命令
- 查看基础组件状态
	- kubectl get componentstatus
	- kubectl get cs
- 查看节点信息
	- kubectl get nodes 
- 查看所有的pod信息
	-　kubectl get pods
	-　kubectl get pods -n kube-system
	-　kubectl get pods -n kube-system -o wide 
- 查看namespace信息
	- kubectl get ns
	- kubectl get namespace   