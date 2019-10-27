##k8s第二课 kubernetes 基础概念

kubernets的集群结构
- master/node
	- master: apiserver ,scheduler, controller-manager，etcd
	- node: kublet, 容器引擎(docker, rkt), kube-proxy
- pod
	- 元数据信息:metadata
	- label: label selector, label表现为key=value格式，定义完成的label可以基于label selector进行筛选

- pod 分类
	- 自主式pod: 自我管理的pod,创建后，由apiserver接收后，借助调度器调度至节点，由节点启动此pod.当pod故障，可以由kubelet进行监控及恢复。当node故障后，无法自动恢复。
	- 由pod控制器控制的pod: 由控制器管理后，pod变成了有生命周期的对象。全生命周期由控制器进行管理。pod控制器有很多，最早的是:
        - Replication controller.控制器指定了需要运行的pod对象个数，必须精确的匹配最初定义的pod数量和状态。
        - 滚动更新／回滚，在更新过程中，可以临时超过期望的pod数量。
        - ReplicaSet
        - Deployment:只能负责管理无状态的应用。同时支持二级控制器HPA（HorizontalPodAutoscaler），可以用于水平扩展的控制器。可以根据pod和系繁忙状态，进行pod数量的水平扩展。
        - StatefullSet: 有状态的副本集
        - DaemonSet: 表现为在每一个节点上的守护进程
        - Job, Ctonjob:单次运行的任务
        - 控制器是为了管理不同类型的pod资源(任务)运行的符合用户期望的状态
	- 用于k8s集群管理和基础功能的pod : Addons附件  

- Service
    - pod生命周期中，有可能在不断的被删除重建，当pod被删除后，新启动的pod中运行的任务是和之前的pod一致，但是同样是一个新的容器。如何保证客户端能正确的接入到这个pod中获取服务。这个需要依赖于服务发现功能。
    - 提供一个服务总线，所有pod提供的服务，都在服务总线上进行注册。用于说明客户端可以在哪里获取到服务。
	- service 的名称和地址都是固定的。service可以交服务请求转发（代理）到后端的具体的pod上。service可以基于pod的标签选择器进行匹配关联。
	- service是iptables 的dnat规则，dnat的地址不是存在任何一个网卡，只是存在在tcp/ip协议栈中。 
	- 地址变更后，组合附件dns服务后，可以实现域名到ip的自动更新。
	- k8s 1.11版本后，iptables dnat规则，现在已使用ipvs替换。可以支持ipvs的各种调度算法
	- service只是用来控制流量转发的
*通过NMT结构，解说service对内，对外暴露的service需求,2个nginx , 3个tomcat , 2个mysql,由物理节点将客户端请求转入到集群中的pod中*

- k8s的网络模型
    - node网络: docker host所在的网络
    - service所在网络(集群网络):与pod网络不同，service地址是虚拟的，只存在iptables或ipvs规则中。
    - pod所在网络:在网络中能ping通
![](./network-struct.png)
- k8s中的通信模型
    - 同一个pod内的多个容器通信:lo
    - 各pod之间的通信:各pod的地址不能冲突，各pod之间可以直接通信。实现方式可以使用直接桥接。使用桥接会造成广播域太大。另外，可以使用overlaynetwork , 可以转发二层报文，可以通过sui道进行三层转发。可以要求host1使用172.17.1.0/24 , 172.17.2.0/24的网络，用于将pod所在的网络进行区分。
    - pod 与service进行通信:报文指向网关即可。会优先匹配iptables规则。kube-proxy用于管理转发规则。每一次service / pod的变化，都需要kube-proxy进行iptables / ipvs的管理信息

- etcd
	- key-value模式的存储
	- 支持restful风格
	- 整个集群的数据都存在etcd中，需要做高可用。
	- 支持https/http，
	- 两个端口，分别需要内部通信和给客户端提供通信
	- 都需要证书支持

- cni
	- 容器网络服务提供接口
	- 所有网络提供商，必须以cni标准来提供网络服务
	- 可以以pod形式运行，共享node节点的网络名称空间
	- flannel:提供网络配置功能，coreos提供
	- calico:提供网络配置，以及网络策略，支持ipip，是一种三层sui道网络
	- calnel: 组合flannel与calico的功能
	- 需要具备基本的网络额离功能
- k8s的名称空间
	- 与docker中的namespace不同
	- 只是逻辑上的隔离，不存在pod中的网络隔离
	- 可以基于cni插件实现k8s namespace空间的隔离，或是同namespace中的pod之间的隔离