## k8s第六课　kubernetes pod控制器应用进阶

资源配置清单：
自主式pod资源
资源清单的格式
一级字段：apiversion(group/version), kind, metadata（name，namespace,labels,annotations）, spec(containers,), status（内嵌很多信息只读，不能修改）
扩展讲apiversion的相关内容，扩展讲kind，资源对象，对象。
道--> 术
当前状态向目标状态不停的迁移

pod资源
spec.containers必备的字段：
kubectl explain pod.spec.containers , 其格式为spec.containers <[]object>
- name 
- image：可以是顶级仓库，可以是用户仓库，也可以是私有仓库，也可以是第三方仓库
关键参数解读: 
- imagePullPolicy: 镜像获取的策略
	- Always: 总是从仓库中下载 
	- Never: 从不下载
	- IfNotPresent: 如果本地不存在，则下载镜像
	- 如果标签值为latest,则默认使用的是Always.如果是特定标签，则默认是IfNotPresent.此选项一被指定，无法再更新。
	- 列举各种情况下获取策略的使用场景。
- ports <[]Object>
	- 端口名称，可以通过名称引用
	- 端口号
	- 使用的协议tcp(default)/udp
	- 容器启动的服务，即使不显示定义端口，在整个容器网络内也是可以访问的，因为容器内使用的是叠加二层网络，但是为自己维护提供便利，可以帮助ops整解有哪些提供的服务。
	- 需要后续测试一下hostIP，hostPort的使用
	- 是一个消息型的配置。
- args
	- 传递给docker image 的entrypoint 的参数
- command
	- Entrypoint array, Not excuted within a shell. 不会以shell形式运行，如果一定要以shell形式运行，需要自己明确指定"/bin/sh", "-c","command"
	- 如果此处不指定，则使用docker image中的entrypoint， 如果指定，则使用当前是的command.
	- 需特别注意，会掩盖entrypoint中的功能，非特殊情况，不建议使用  

修改镜像中的默认命令，通过command, args来设置，command and args的生效矩阵

|Image Entrypoint| Image cmd| container command| container args| command run|
|:---:|:---:|:---:|:----:|:---:|
|/ep-1| [foo bar]| not set | not set | [ep-1 foo bar]|
|/ep-1| [foo bar]| ep-2 | not set | [ep-2]|
|/ep-1| [foo bar]| not set | [zoo boo] | [/ep-1 zoo boo]|
|/ep-1| [foo bar]|ep-2| [zoo boo] | [ep-2 zoo boo]|

如果pod中只有一个容器，则直接使用kubectl logs pod-name就可以， 如果有多个容器，则需要使用-c container-name 来指定需要输出pod内哪个容器的日志。
标签是附加的对应的对象上的键值对。一个资源上可以存在多个标签。 每一个标签都可以被标签选择器进行匹配度检查。一个标签可以用在多个资源对象上。 资源对象和标签是多对多的关系。 
- 标签可以在创建时指定
- 也可以通过命令在资源创建后修改和管理
- 标签一般可以从几个维度进行区分
	- 按照分层应用架构中，所属的层次. tier: frontend/cache/datastore/application
	- 版本标签: release: cannary, v1,v2, betal1
	- 环境标签: dev/test/prod 
	- 应用程序自身特性: app: tomcat/php/rails/python
	- 查看标答: kubectl get pods --show-labels
	- key=value , 键名，值不能超过63个字符，key只能使用字母／数字／下划线／中横线／.号组成。value可以为空，其命名格式与key相同。 只能以字母或数字开头及结尾。
	- key,value要做到见名知义
	- 通过标签过滤显示：kubectl get pods -L app=myapp
	- kubectl get pods -L app　, -L --label-columns= , 以列的形式对标签进行分组显示，而非进行标签过滤
- 标签选择器
	- 等值关系的标签选择器: =, ==, != 
		- kubectl get pods -l app=myapp
		- kubectl get pods -l app=myapp,release=canary
		- kubectl get pods -l release!=stable
	- 集合关系
		- KEY IN (VAL1,VAL2) 
		- KEY notin (val1,val2)
		- key
		- !key
		- kubectl get pods -l "release in (canary,alpha,beta1)"
		- kubectl get pods -l "release notin (canary,alpha)"
	- 空集合：表示所有对象都符合	
		- kubectl get pods -l app , 对标签进行筛选，只显示含有app为标签key的pod 
		- 
- yaml清单资源中的标签选择器
	- matchLables：直接给定键值
	- matchExpressions:基于给定表达式使用标签选择器。{key:"KEY", operator:"Operator:In/NotIn/Exists/NotExists", values:[VAL1,VAL2,VAL3]}
        - operator : In/NotIn, values必须为非空列表
        - operator: Exists, NotExists ,values必须为空列表
- 修改资源标签
	- kubectl label TYPE NAME KEY_1=VALUE_1 KEY_2=VALUE_2
	- kubectl label pod pod-demo release=canary
	- kubectl label pod pod-demo release=stable --overwrite
	- 同样支持对节点进行打标签：kubectl label node node01 disktype=ssd
	- 当节点有标签后，则可以设置pod与节点之前的亲向性，可以基于nodeSelector进行节点标签的筛选

-L 显示效果说明
~~~ bash
[root@k8smaster ~]# kubectl get pods --show-labels
NAME       READY   STATUS    RESTARTS   AGE    LABELS
pod-demo   2/2     Running   0          142m   app=myapp,tier=frontend

[root@k8smaster ~]# kubectl get pods -L app,tier
NAME       READY   STATUS    RESTARTS   AGE    APP     TIER
pod-demo   2/2     Running   0          142m   myapp   frontend

~~~

####将pod运行在特定的node上
- nodeSelector <map[string]string> 节点标签选择器
可以使用nodeSelect 进行节点的固定，或是使用nodeName直接将pod固定在某个节点上。　
将某个节点打上标签：　 kubectl label node k8snode1 disktype=ssd
[root@k8smaster manifests]# kubectl label node k8snode1 disktype=ssdserver
error: 'disktype' already has a value (ssd), and --overwrite is false
[root@k8smaster manifests]# kubectl label node k8snode1 disktype=ssdserver --overwrite
node/k8snode1 labeled

~~~ bash
[root@k8smaster manifests]# cat myapp.yaml 
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
    ports:
    - name: http
      containerPort: 80
      protocol: TCP
    - name: https
      containerPort: 443
  - name: busybox
    image: busybox
    command:
    - "/bin/sh"
    - "-c"
    - "while true; do echo $(date) >> /usr/share/nginx/html/index.html && sleep 5; done" 
  nodeSelector:
      disktype: ssdserver
[root@k8smaster manifests]# kubectl label node k8snode1 disktype=ssdserver
error: 'disktype' already has a value (ssd), and --overwrite is false
[root@k8smaster manifests]# kubectl label node k8snode1 disktype=ssdserver --overwrite
node/k8snode1 labeled
[root@k8smaster manifests]# kubectl get pods 
NAME       READY   STATUS    RESTARTS   AGE
pod-demo   2/2     Running   0          7m25s
~~~

label完成后，立即变成running状态。如果未打标前，则提示无可调度的节点。　
~~~ bash
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  104s (x63 over 6m46s)  default-scheduler  0/2 nodes are available: 2 node(s) didn't match node selector.

~~~

- nodeName <string>

annotations: 与label的不同之处在于，它不能用于挑选资源对象，仅用于为对象提供“元数据”。键值不受长度限制。
annotation属于metadata字段，使用格式一般是
组织名／键名：值
eg. magedu.com/author: Paul.Lai
~~~ bash 
metadata: 
  name: pod-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
  annotations: 
    magedu.com/created-by: Paul.lai

~~~

pod 生命周期
秒级启动，是指启动容器的时候。而容器内的程序进行初始化，或是由entrypoint进行初始化，需要花费一定的时间。所以秒级启动对docker来说不具有特别大的意义。　

状态：　
pending (pod在调度中，已经创建，但是没有适合它运行的节点。　此时pod并未启动)
running: 运行状态
failed: 失败
successed; 
unknow: api server 与kubelet通信，并且返回状态。如果kubelet出错，则会提示类似Unknow的状态
创建pod :  apiserver ,将目标状态保存在etcd中，然后会请求scheudler，调度完后，结果保存在etcd中。　被调度的目标节点，通过kubelet从apiserver中拿到相应的任务和状态，则在本地节点上创建西相应的状态。schudler负责挑选出合适的节点。
![](./pod-lifecycle.png)

存活性探测主要用于判断主容器是否正常运行。
就续性探测主要用于判断主容器的进程是否正常提供服务，　比如http的状态码正常返回200. 

pod 生命周期中的重要行为：　
- 初始化容器
- 容器探测
	- liveness:　容器是否正常运行
	- readiness: 服务是否正常
- post start 
- 容器运行
- pre stop 

容器的重启策略：
pod仍然存在，　但是容器挂了。　
- restartPolicy
	- One of Always, OnFailure,Never. Default to Always 
	- Always: 一旦pod中的容器挂了，第一次重启，将立即重启container；第二次重启，将先等待10s, 之后是20s, 之后是40s, 最大的等待间隔是300s.　
	- OnFailure: 只有在失败状态时才重启，即liveness探测失败时。

一旦一个节点被调度到某个节点上，　那么它将会一直运行在同一个节点上。除非重建pod或是所运行的节点故障，或是标识为维护。
pod是运行服务的主要单位。
删除过程：
pod删除时，会给一个kill -15信号term，需要容器内的进程发送信息，如果超过中止时间， 之后发送kill -9 .强行进行中止。 宽限期默认是30s.也可以自定义时间。  