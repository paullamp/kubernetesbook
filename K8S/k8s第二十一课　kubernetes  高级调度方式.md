# 第二十一课　kubernetes高级调度方式

可以通过预设的操作，用于影响pod的调度。因为普通情况下的直接使用系统默认的调度方式，可能与我们的实际期望有一定的差异。比如我们期望某一部分的POD运行在哪个区域，运行在哪些类型的服务器上，或是运行在哪个部门所属的独立服务器上。

高级调度设置方式主要包括三种

## 节点选择器

- nodeSelector

  当期望pod运行在一组具有特定标签的节点上时。

  kubectl explain pod.spec.nodeSelector

- nodeName

  明确指定pod运行在哪个节点上。需要提供node的名称

  

  使用nodeSelector的示例：

  ```yaml
  apiVersion: v1
  kind: Pod
  
  metadata:
    name: pod-scheduler-demo
    namespace: default
  spec:
    containers:
    - name: pod-container
      image: nginx:1.16.1-alpine
      imagePullPolicy: IfNotPresent
    nodeSelector:
    	ssd: "on"
  ```

  注意：　ssd 后面的　“on”, 必须用引号引用起来，不然在dry-run的时候不出错，但是在实例执行的时候会提示如下的错误。

  ```
  Error from server (BadRequest): error when creating "pod-demo.yaml": Pod in version "v1" cannot be handled as a Pod: v1.Pod.Spec: v1.PodSpec.NodeSelector: ReadString: expects " or n, but found t, error found in #10 byte of ...|":{"ssd":true}}}
  |..., bigger context ...|esent","name":"nginxpod"}],"nodeSelector":{"ssd":true}}}
  ```

  当首次调度失败后，scheduler会在一定周期后进行重试。直到三次重试都失败后，会返回失败。　nodeSelector在预选调度中是强约束。如果无法匹配成功，则不进行后续的调度。

##　节点亲和性

kubectl explain pod.spec.affinity

```
FIELDS:
   nodeAffinity	<Object>
     Describes node affinity scheduling rules for the pod.

   podAffinity	<Object>
     Describes pod affinity scheduling rules (e.g. co-locate this pod in the
     same node, zone, etc. as some other pod(s)).

   podAntiAffinity	<Object>
     Describes pod anti-affinity scheduling rules (e.g. avoid putting this pod
     in the same node, zone, etc. as some other pod(s)).
```

其中包含了三种类型的亲和性调度控制对象。

- nodeAffinity: 节点亲和性调度

  根据调度过程影响的强制性，有两个可选的参数。　

  - preferredDuringSchedulingIgnoreDuringExecution: 软亲和性。期望能够匹配此节点亲和性调度，如果不能匹配上，也会将pod调度到此节点上。　获得最大评分的节点将被选中。尽量满足。
  - requiredDuringSchedulingIgnoreDuringExecution：硬亲和性。必须满足匹配要求，对不满足要求的，一定不会调度到此节点上。　

- podAffinity: pod亲和性调度

- podAntiAffinity:　pod反亲和性调度



nodeAffinity-required的示例：

```yaml
apiVersion: v1
kind: Pod

metadata:
    name: pod-node-affinity-pod
    namespace: default
    
spec:
    containers:
    - name: nginx-pod
      image: nginx:1.16.1-alpine
      imagePullPolicy: IfNotPresent
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            #- matchFields:
            #    - key: "ssd"
            #      operator: Exists
            - matchExpressions:
              - key: zone
                operator: In
                values:
                  - foo
                  - bar
```



nodeAffinityPerferred

 pod-node-affinity-perferred-demo.yaml

```yaml
apiVersion: v1
kind: Pod

metadata:
    name: pod-node-affinity-perferred-demo
    namespace: default

spec:
    containers:
    - name: nginx-pod
      image: nginx:1.16.1-alpine
      imagePullPolicy: IfNotPresent
    affinity:
      nodeAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
            - preference:
                matchFields:
                - key: ssd
                  operator: Exists
              weight: 60

```

后面需要重点排查一下matchFields.key如何进行匹配。上面的这个配置文件会报错。



##　pod亲和性

pod之前会有亲向，运行在同一个节点上。　比如一个nginx-tomcat-mysql结构，他们会倾向于部署在同一个NODE. 而对于多个tomcat ,因为考虑到冗余结构，则可能会倾向于不运行在同一个node. 



使用节点亲和性不是一种较优的方式，而且会依赖于node标签。　

允许第一个pod随机选择一个位置，而后其他的pod会依据这个进行相对定位。　自己可以先规划一下哪些pod要亲和，哪些pod要反亲和。

什么是同一个位置，什么是不同位置？

比如以节点名称为标准，则一个节点是一个位置。或者把某个标签做为位置判断标准，则拥有相同标签的是属于同一位置。



labelSelector: 用于筛选需要进行亲和匹配的pod资源.下面有两个属性，matchExpressions和matchLabels。

- matchExpressions: 集合选择器
- matchLabels: 标签选择器

以上两上选择器，与之前的pod控制器所需的选择器相同的使用方法。

namespaces：pod所运行的名称空间列表。如果没有指定，则表示与当前pod相同的名称空间。一般而言，不会跨名称空间去引用pod.

topologyKey:  node的拓扑组成结构。



定义硬亲和pod

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: pod-nginx-pod-affinity-required-demo
  labels:
  	app: homewebsite
  	layer: input
spec:
  containers:
    - name: nginx-con
      image: nginx:1.16.1-alpine
      imagePullPolicy: IfNotPresent
      
--- 
apiVersion: v1
kind: Pod

metadata:
  name: pod-tomcat-pod-affinity-required-demo
  labels:
    app: homewebsite
    layer: app
spec:
  containers:
  	- name: tomcat-con
  	  image: tomcat:v1
  	  imagePullPolicy: IfNotPresent
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoreDuringExecution:
      - labelSelector:
          matchExpressions:
            - {key: app, operator: In, values: ["homewebsite"]}
         topologyKey: kubernetes.io/hostname
```

pod反亲和性只是要求最终结果与亲和的结果相反，即pod不运行在同一个node节点上.



##　污点及容忍调度

污点是一个键值属性数据。现在有三类键值数据，分别为label , annotation ,taints .

label，annotation是可以应用于所有的资源类型，taints只能应用于节点。　

污点主要是用于来拒绝pod. 拒绝所有不能容忍的污点。



kubectl taints node01 

kubectl explain node.spec.taints

```
   effect	<string> -required-
     Required. The effect of the taint on pods that do not tolerate the taint.
     Valid effects are NoSchedule, PreferNoSchedule and NoExecute.

   key	<string> -required-
     Required. The taint key to be applied to a node.

   timeAdded	<string>
     TimeAdded represents the time at which the taint was added. It is only
     written for NoExecute taints.

   value	<string>
     Required. The taint value corresponding to the taint key.

```

污点属性的关键信息：

effect: NoSchedule,  PerferNoSchedule, NoExecute  分别表示不参与调度，　不建议调度，不调度并且如果调度成功，需要进行驱逐。

taints 的effect定义node对pod排折效果（等级）：

- NoSchedule: 只影响调度过程，不容器，不会调度至此节点。对现存节点不产生影响
- NoExecute: 　不仅影响调度，而且影响现存的pod对象。不容忍的则被驱逐
- PerferNoSchedule: 　最好不要调度到这个节点

### 设置pod的污点容忍度

pod的容忍度可以是等值比较或是包含比较。

声明pod可容忍的污点示例；

```yaml
apiVersion: v1
kind: Pod

metadata:
    name: pod-taints-demo
    namespace: default

spec:
    containers:
      - name: nginx-pod-tolleration
        image: nginx:1.16.1-alpine
        imagePullPolicy: IfNotPresent
    nodeName: an2
    tolerations:
        - key: node-role.kubernetes.io/master
          operator: "Exists"
          value: ""

```

不指定effect时，则表示会容忍所有的effect

###　管理节点上的污点

标识为生产环境专用的node

```
kubectl taint node node01 node-type=production:NoSchedule
```

其中key=node-type, value=production , effect= NoSchedule

如果修改了节点的容忍度，则它会被重新调度一次。

