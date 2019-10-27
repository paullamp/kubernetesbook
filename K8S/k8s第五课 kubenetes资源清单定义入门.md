## k8s第五课　kubernetes资源清单定义入门

k8s提供了restful风格的api，支持最期本的GET,PUT,DELETE,POST增删改查操作。对应于kubectl命令，有很多的命令可以实现curd操作，例如：kubectl run, get ,edit, expose。kubectl操作的是所有的k8s集群内的资源对象。

### kubernetes 的常用资源(对象)

资源分类可以分为以下几个大类:

- 工作负载型(workload)资源对象
    - pod
    - pod控制器: replicaset, deployment, statefulset,daemonset, job, cronjob, 
- 服务发现及服务均衡
    - service
    - ingress
- 配置与存储相关
    - Volume
    - CSI:容器存储接口，扩展第三方接口
    - ConfigMap
    - Secrect
    - DownwardAPI
- 集群级资源
    - namespace
    - node
    - role
    - clusterrole
    - rolebinding
    - clusterrolebinding
- 元数据型资源
	- HPA
	- PodTemplate
	- LimitRange


### 清单文件定义
使用配置清单来创建相应的资源。使用`kubectl get pod pod-name  -o yaml`命令，可以输出一个yaml格式的资源清单。资源清单是用户用于管理k8s对象的定义工具。在清单中定义了用户期此对象所处的目标状态。创建资源的方法：只接收json格式的资源定义；json格式太重，yaml格式更加简洁。默认以yaml格式提供配置清单。apiserver可以将其自动转换成json格式，后再提交至apiserver。

大部分资源的配置清单，都有以下几个重要部分组成：　
- apiVersion：　
	- groupname/version，如果groupname未出现，则表现使用core组，或app组.
	- kubectl api-versions
- kind
	- 资源类别
- metadata
	- name:同一类别当中，名称必须唯一，对等于类的具体的对象
	- namespace: 同名称空间中，name须唯一
	- labels 
	- annotations:注解
- spec
	- 用于描述用户期望的状态，desired state
	- 不同的资源类型，所需要的spec不同。　
	- 标识为required的，为必选字段， 
- status:
	- 目标对象的当前状态，本字段由kubernetes进行维护，用户不能修改
	- k8s会使用status无限趋近spec指定的期望状态。

查看不同字段的帮助信息
kubectl explain pod 
kubectl expalin pod.metadata
kubectl explain pod.spec

每个资源的引用路径：
/api/GROPUNAME/API-VERSION/namespaces/NAMESPACE-NAME/TYPE/INSTANCE-NAME

api分组的好处，分组后，方便后期的更新，如果只是一个组的功能更新，只更新一个功能即可。加上版本后，可以让同一组的多个版本共存。　
alpha内测版 , beta 公测版, stable稳定版

所有的map对象，可以使用{}进行编辑，所有的列表，可以使用[]进行连接。

一个pod-demo.yaml格式的pod示例:
~~~ bash
apiVersion: v1
kind: Pod

metadata: 
  name: pod-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend

spec:
  containers:
  - name: mynginx
    image: mynginx:v1
  - name: busybox
    image: busybox
    command:
    - "/bin/sh"
    - "-c"
    - "while true; do echo $(date) >> /usr/share/nginx/html/index.html && sleep 5; done" 

~~~
其他写法：
- 列表格式：其中command是列表格式的对象。　可以改写成如下的一行:`command: ["/bin/sh", "-c", "while true;do echo ...;done"]`

- 字典格式map: `labels: {app:myapp, tier:frontend}`

诊断pod的状态
- kubectl logs pod-demo mynginx
- kubectl logs pod-demo busybox
- kubectl exec pod-demo -c busybox -- /bin/sh
- kubectl delete -f pod-demo.yaml
- kubectl create -f pod-demo.yaml

三种用法进行k8s资源管理:
- 命令行创建资源: 基于命令行式进行操作，类似于kubectl run, kubectl expose等命令。
- 命令式资源清单：可以重复使用资源清单，资源清单的变更无法再使用命令式创建
- 声明式资源清单:可以直接修改资源清单，并且再次声明。资源会向定义的期望驱近。