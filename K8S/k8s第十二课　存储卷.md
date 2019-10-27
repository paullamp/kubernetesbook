# k8s 第十二课　存储卷

k8s里运行的pod，按着是否有状态，以及是否需在持久存储，可以分为四个象限。k8s里的应用，按照状态和数据的维度，可以分为有状态、无状态，按照是否需要数据存储，可以为需要持久存储，以及不需要持久存储。共四个维度的pod应用类型。 针对需要持久存储的应用，则需要使用volumes来存储数据。

容器本身有生命周期，也就是容器的，需要持久存储的数据必须放在容器外的一个共享空间中。

当pod被删除时，pod内容器的数据也会清理，并且，pod有可能会离开原先运行的node。数据有可能会随着pod的终结而终结。因此，需要将pod内的数据放到pod自有空间之外。　

共享存储设备来给k8s的pod提供空间。　k8s提供了各种类型的存储卷以提供持久存储。

同一个pod内的多个容器，可以共享同一个存储卷。k8s的存储卷是属于pod，而不是属于容器，　k8s.gcr.io/pause是一个基础架构容器，运行在pod内的容器会共享pods内的网络名称空间、存储卷。

脱离节点本地的存储空间，而要使用网络存储空间。

k8s可用的存储卷：

- emptydir: 临时目录，或是缓存目录。可以是宿主机的目录，也可以是内存当作硬盘使用。
- hostpath: 主机目录，相当于docker中的volumes
- 网络存储：
  - san/nas：　传统存储，　常用的nas协议nfs,cifs.san中常用的协议有iscisi
  - 分布式存储：　文件系统存储级别，或是块级别的。　glusterfs,cephfs, ceph rbd
  - 云端存储：　ebs, azure block. k8s运行的公有云服务上时。可以直接对接云存储

PVC : persistentVolumeClaim,相当于在存储pv与pod之间再抽象一层。pod与底层存储解料。pvc是与pv存在关系。

用户可以不用关注pv,以及pv 与底层存储的关系，配置，比如与底层存储需要的认证信息，连接，目录。　另外，也可以隐藏底层存储的具体实现。　

存储类：可以动态供给存储空间

pod可以直接使用底层的存储空间。也可以使用pvc而实现动态供给。

使用存储卷的步骤：　

pod上定义volume

容器中使用volumeMount，才能使用

```bash
kubectl explain pod.containers.volumes
kubectl explain pod.containers.volumeMounts
```

小实验：创建一个自主式的pod使用emptyDir类型存储卷.

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: testEmptyDir
  namespace: default
spec:
  containers:
  - name: testEmptyDir-con
    image: mynginx:v1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: emp
      mountPath: /mnt
      readOnly: false
  volumes:
  - name: emp
    emptyDir:
      media: ""
      sizeLimit: 50Mi
```

关于busyboxpod的command使用：

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: mypodtest
  namespace: default

spec:
  containers:
  - name: b1
    image: busybox
    imagePullPolicy: IfNotPresent
    command: 
    - /bin/httpd
    - -f
    - -h
    - /tmp
  hostAliases:
  - hostnames:
    - code.grandsoft.com.cn
    ip: 192.168.0.201

```

> 注意，command每一个参数必须分隔开。比如"-h /tmp/"不能合在一起写，否则会出错。同时可以支持列表格式command: ["/bin/httpd", "-f", "-h", "/tmp/"] . 如果使用command: ["/bin/httpd", "-f", "-h /tmp/"]则会出错。

小实验：　测试同一个pod内的多个容器，共享同一个数据目录

shareemptydir.yaml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: sharedir
  namespace: default

spec: 
  containers:
  - name: maincontainer
    image: mynginx:v1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: empdir
      mountPath: /usr/share/nginx/html
    
  - name: busyboxsidecar
    image: busybox
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: empdir
      mountPath: /usr/share/nginx/html
    command: ["/bin/sh","-c", "while true; do date >> /usr/share/nginx/html/htllo.html && sleep 5;done"]
  
  volumes:
  - name: empdir
    emptyDir: {}

```

> 不能使用$(date)替代date, 因为$(date)时，程序会将date的输出当成命令执行。而date输出的日期不是正常的命令。

> emptydir的生命周期与pod生命周期一致，因此，　emptydir无法持久存储数据。

### gitRepo 存储卷

会自动从git仓库中获取内容，并且将内容挂载到存储卷中。gitRepo是构建在emptyDir的基础上。它是相当于做了一次git 初始化的emptyDir. 创建一个emptydir,  然后再执行git .它只取了clone那一时刻的代码。当git仓库更新了，gitRepo存储卷的内容不会改变。

### hostPath

将宿主机的某个目录，当成一个存储卷挂载给容器中。此存储卷的生命周期不会随着容器消失。缺点是只能局限于在单个主机上的POD,可以做数据共享。　

hostpath-pod.yaml 

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: hostpathpod
  namespace: default

spec:
  containers: 
  - name: hostpath-container
    image: mynginx:v1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: hostpathdir
      mountPath: /usr/share/nginx/html

  volumes:
  - name: hostpathdir
    hostPath: 
      path: /data/hellohostpath
      type: DirectoryOrCreate
```

> pod运行在哪个node上，则对应的path: /data/hellohostpath就会创建在哪个主机上。kubectl get pods -o wide ，可以简易查看pod　ip和所在的　node. 此时可以使用nodeName ,将pod固定在某个特定的node上。

只能提供节点级的持久。

###　NFS

部署nfs-server

yum install nfs-utils

nfs 导出/etc/exports：　/data/volumes 172.20.0.0/16(rw,no_root_squash)

监听　2049

systemctl start nfs.service



node 节点需要能支持nfs挂载，如果不支持，则需要安装nfs-utils.

可以在node上尝试挂载, mount -t nfs-server:/data/volumes /mnt, node必须能挂载目标目录。

小实验: 测试使用nfs作为存储卷

```yaml
apiVersion: v1
kind: Pod

metadata: 
  name: nfs-pod
  namespace: default

spec:
  containers:
  - name: nfs-pod-c
    image: mynginx:v1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: purenfs
      mountPath: /data/
    
  volumes:
  - name: purenfs
    nfs:
      path: /data/nfs_server/purenfs
      readOnly: false
      server: 10.1.89.234

```

> NFS没有冗余能力，建议使用自建的分布式存储系统，如glusterfs , ceph. ceph可以直接提供restful接口。

小实验：尝试使用deploy控制器来管理共享有nfs的pod. 

如果要使用nfs ,那么需要知晓后端的存储的连接信息。　如路径，服务ip等。因此，可以将存储功能剖离开。

### PV & PVC

pv & pvc的结构如下：　

![pod-pvc-pv-store](pod-pvc-pv-store.png)

pv,pvc是k8s上资源的一种抽象，它是一种标准资源。

存储工程师负责处理后端的存储维护。pvc假定需要5g空间，则会自动在pv上资源识别。会选择最合适的pv。

![1565277001226](E:\k8s+docker\notebook\K8S\pv-ops.png)

kubectl explain pvc

accessModes: 是否允许多人访问

resources: 允许多少资源，比如说至少允许10G

pvc与pv是一对一关联的。不可以一对多关联。如果某个pv被一个pvc占用了，那么这个pc将无法被别的pvc申请。一个pvc可以被多个pod访问。

当pvc与pv关联后，则pv的状态是binding. 

nfs操作命令： exportfs -avr 

showmount -e 

nfs格式的pv: 



```yaml
apiVersion: v1
kind: PersistentVolume

metadata: 
  name: p-pv1
  namespace: default

spec: 
  capacity:
    storage: 500Mi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /data/nfs_server/v1
    readOnly: false
    server: 10.1.89.234
---
apiVersion: v1
kind: PersistentVolume

metadata:
  name: p-pv2
  namespace: default
spec:
  capacity:
    storage: 100Mi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /data/nfs_server/v2
    readOnly: false
    server: 10.1.89.234

```

> pv是集群级的资源，不能定义在名称空间中。pv资源可以在全部的名称空间中使用。

参考文档：https://kubernetes.io/docs/concepts/storage/persistent-volumes#access-modes

不同的存储卷，可以支持不同的accessModes . accessModes必须是后端存储能力的子集。不可能是后端的能力超级。　有以下三种存储读取能力。　ReadWriteMany, ReadWriteOnce, ReadOnly

capacity: 只有一个可以使用的存储资源，即storage,用于指明可以供使用的存储空间大小。

Reclaim Policy : 回收策略。pv,pvc之间的绑定解除后，如何处理　pv内的数据。　retain表示数据仍然保留　，recycle：回收，　数据会清理。　delete : pvc消亡的时候，pv也会清理。　以后两种情况要慎用。

kubectl explain pvc.spec.resources

pvc的定义：　

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: p-pvc1
  namespace: default

spec: 
  accessModes: 
  - ReadWriteMany

  resources: 
    requests:  
      storage: 100Mi
```

小实验: pod 引用pvc.

当pod删除时，pv, pvc能正常使用。数据不丢失。

集群状态存储etcd. 

注意pv的回收策略非常关键。

现在只要pv还被pvc 绑定，则pv不可被删除。当pv,pvc pending时，pod是无法创建的。

pod可以使用class实现动态供给。