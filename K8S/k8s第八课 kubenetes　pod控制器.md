## k8s第8课 kubernetes pod控制器

回顾: pod
apiVersion: 
kind: 
metadata: 
spec
status: 这个在定义时不需要指定， 是一个只读字段。 
spec: 
  containers: 
  name
  image
  imagePullPolicy: Always, Never, IfNotPresent
  ports: name, containerPort|
  livenessProbe:
  readinessProbe:
  lifecycle
  	EXecAction: exec
    TCPSocketAction: tcpSocket
    HttpAGetAction: httpGet
  
  nodeName
  nodeSelector:
  restartPolicy:

### pod控制器

自主式pod在被删除后，将不会再重建。 由控制器进行管理的资源，控制器可以管理pod资源符合用户期望的数量。 

pod控制器有多重类型 
replicationController:
replicaset: 代用户创建一定数量的副本，可以支持滚动更新，自动扩缩容。
包含三个部分： 用户期望的副本数， 标签选择器， pod模板.
kubernetes不建议使用rs控制器，而建议使用deployment
deployment是建立在replicaset 上的，而非直接建立在pod上的。 支持rs的功能，而且支持滚动更新和回滚。声明式定义。 
一个节点可以运行一个deploy的多个pod副本。

daemonSet: 保证每一个node只运行一个特定的pod副本， 一般用于实现系统级的后台任务。只要新增加节点，相应的pod是自动增加。 包含 标签选择器， pod模板。因为每个节点都只运行一个， 所以数量不用指定。 有标签选择器的也是一样， 可以先匹配标签， 然后根据标签筛选出来的结果，每个运行一个。 

deploy , deamonset 有以下特点： 
无状态应用（只关注群体，不关注个体)
需要持续运行及监听

Job: job是否需要重建， 主要看任务是否完成。 适合是一次性的作业。 
cronjob: 周期性的任务

StatefullSet: 管理有状态应用，拥有独立的标志和数据集。提供了一种封装。处理逻辑是需要运维的人来写。需要设计各种可能的应用场景和故障场景。比如说数据的恢复过程，截取的数据位置。 有状态的运维操作步骤会比较麻烦。


TPR资源： Third Party resources , 
CDR资源： custom Defined Resources , 1.8 + 会面换成CDR , 不再支持TPR.

Operator: 运维集合，比statefullset 更简单。
k8s的使用门槛较高。 
helm 使用helm 降低了使用难度， 类似于rpm , yum 等工具，可以安装安装。 helm还比较新。


redisCluster: 

replicaSet的配置及实验：
~~~ bash 
[root@k8smaster controllers]# cat demo-rs.yaml 
apiVersion: extensions/v1beta1
kind: ReplicaSet

metadata: 
  name: myapp-rs
  namespace: default

spec: 
  replicas: 2 
  selector: 
    matchLabels: 
      appname: myapp
  template: 
    metadata:
      name: myapp-pod
      labels:
        appname: myapp
    spec: 
      containers: 
      - name: myapp-container
        image: mynginx:v1
        ports: 
        - name: httpd
          containerPort: 80     
[root@k8smaster controllers]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
myapp-rs-lxph2   1/1     Running   0          22s
myapp-rs-r9pzh   1/1     Running   0          22s
[root@k8smaster controllers]# kubectl get rs 
NAME       DESIRED   CURRENT   READY   AGE
myapp-rs   2         2         2       24s
[root@k8smaster controllers]# 

~~~
在pod模板中起的名字没有用， 它会根据pod控制器中的内容进行重命名。 
kubectl get pods --show-labels
注意，pod的标签在控制器中起着重要的作用，所以建议使用复杂条件，以避免出现pod冲突。
针对两个pod，最好使用一个前端service进行高度。 

service 与控制器没有关联关系，只与标签选择器有关系。 service 可以使用控制器创建pod作为后端。 
rs 最大的功能，是支持动态的扩缩容。 
扩容数量，可以直接使用kubectl edit rs myapp-rs ，可以修改 replicas: 2， 

kubectl get rs -o wide 
kubectl scale 或是直接使用编辑配置文件。 
只有重建出来的镜像是新的， 之前留存的docker进程仍然使用旧的镜像。 
可以逐个删除原来的docker 实例，或是一次全部删除。删除重建后的数据，将全部替换成新的镜像id.
同一套service , 可以后端挂两个rs , rs1,rs2分别是两个版本， 当rs2更新了，则将可以将rs1去除。 


一个deployment可以管理多个不同版本的rs .可以自定义需要保留多少个版本， 并且控制更新期间的策略。 比如允许最多的pod个数。 如果需要更新， 可以使用kubectl patch方式进行更新。 控制更新粒度。（允许多一个，不允许少； 允许少，不允许多；或是控制上下限）在此场景下livenessProbe, readinessProbe特别重要。 

