# k8s第二十课　调度器、预选策略及优选函数

在k8s中能运行pod 就是我们的node节点，也就是工作节点。　

master只是控制平台，它不运行工作负载。工作负载一般只会运行node节点上。　



scheduler调度器用于控制pod被调度到哪个节点。

scheduler 高度器需要决策将POD放到哪个最佳的节点上。

scheduler调度分为三级（三步）操作。

## 预选策略(predicate)

预选策略：　从所有的nodelist中，排除不符合自身的运行要求的节点。

资源需求：起始资源基本要求

资源限制：超过这个限额，将不再分配任何内存及ＣＰＵ资源

当前占用：当前ＰＯＤ使用了多少资源

如果有特殊偏好，可以根据特殊偏好来运行。nodeSelector , nodeName. 

nodeselector可以用于匹配节点标签

nodename  是强制使用某个节点

调度函数：　

节点亲和性nodeaffinity，

pod亲和性 pod affinity, pod运行在同一个位置，如果不想运行在同一个ＮＯＤＥ，则是反亲和性。

污节以及污点容忍度：Taints （节点级别）, Tolerations (pod级别)

容忍期：当pod不能容忍ＮＯＤＥ的污点时，后续的处理动作在什么时候执行。　

通常是做均衡调度。

调度算法：[kubernetes](https://github.com/kubernetes/kubernetes)/[pkg](https://github.com/kubernetes/kubernetes/tree/master/pkg)/[scheduler](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler)/[algorithm](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler/algorithm)/[predicates](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler/algorithm/predicates)/**predicates.go**

###　预选函数  

一票否决

CheckNodeCondition：　节点状态是否正常

GeneralPredicate:

​	HostName: pod.spec.hostname 检查pod对象是否定义了

　PodFItsHostPorts:  pod.spec.containers.ports.hostPort 指定要绑定到某个节点上的哪个端口上

​	MatchNodeSelector: pod.spec.nodeSelector

​	PodFitsResources : 检查节点是否有足够的支持运行pod的最低限制(request), pod.spec.containers.resources.requests.每一个节点都会声明自己节点可以被使用的资源量。通过kubectl describe nodename , 可以查看对应的Capacity, Allocatable两个字段，里面有node可以提供的能力，以及已经被使用的情况

NoDiskConfict :是否满足存储卷需求。默认未启用

PodToleratesNodeTaints: 检查pod上的pod.spec.tolerationss可容忍的污点是否宛全包含节点上的污点。

PodTolerateNodeNoExecuteTaints: noexecute,不再容忍的时候，会交pod驱逐出此ＮＯＤＥ。　默认不检查

CheckNodeLabelPresence: 检查标签存在性，　默认不启用

CheckServiceAffinity: 根据当前pod对象所归属的service，将相当service的pod对象放在同一个ＮＯＤＥ,使得内部需要通信息ＰＯＤ的通信成本降低。默认未启用。　

CheckVolumeBinding: pvc相关，　pvc是否满足需求

NoVolumeZoneConflict : 

CheckNodeMemoryPressure: 检查节点内存资源是否存在压力。

CheckNodePidPressure : 检查节点上的pid数据是否存在压力。

checkNodeDiskPressure: 检查节点上的磁盘是否存在压力

MatchInterPodAffinity: 检查是否满足pod亲和性和的亲和性



所有的预选策略都需要全部匹配一次，一票否决策略。



​	



## 优选策略(priority)

为每一个节点进行评分，选取评分最高的ＮＯＤＥ



###　优选函数

得分相加，取分值最高，如果得分一样，随机选择一个。

代码位置：[kubernetes](https://github.com/kubernetes/kubernetes)/[pkg](https://github.com/kubernetes/kubernetes/tree/master/pkg)/[scheduler](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler)/[algorithm](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler/algorithm)/**priorities**/

least_requested.go 根据空闲的比率

cpu :  (capacity-sum(requested))*10 / capacity , 每个优先函数的分值为10分。乘以占用比例　

mem: (capacity-sum(requested))*10 / capacity

得分最高的，则为最优节点。　



BalancedResourceAllocation: cpu和mem率的占用均衡比例超相近，评分越高。



NodePerferAvoidPods: 节点不要运行相应的pod, 优先级极高。根据节点的注解信息(annotations), sechuduler.alpha.kubernetes.io/perferAvoidPods . 



TaintToleration: 将pod对象的spec.tolerations 列表项与节点的taints列表项进行匹配度检查，匹配条目越多，得分越低。　基于pod对象spec.tolerations, 匹配越多，表示分值越低。

SelectorSpreading: 尽量使用pod分散到更多的节点上。即含有同样标签的pod会分散到不同的node上。　有越多要同标签的pod得分越低。

InterPodAffinity:　亲和性匹配项越多的，得分越高。　

NodeAffinity: 节点亲和性，与pod的节点亲和性进行匹配。

MostRequested: 　空闲量越小的节点，越优先被调度。　尽量把节点调度集中到已使用的node上。　默认不启用。

NodeLabel : 根据是否拥有标签。有标签即可得分。默认不启用。

ImageLocality:  判断节点上是否有拥有需要运行容器的镜像。匹配的镜像越多，得分越高。根据镜像的体积大小评估。　镜像越大，所需要的下载时间越长。　默认不启用。





##　绑定(select)

如果评分一样的，则会随机选择一个。　



预选、优选时，可以根据自己定义的选择方式，来让ＰＯＤ按我们期望的目标进行转移。　