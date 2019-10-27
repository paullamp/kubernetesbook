## k8s第9课 kubernetes Pod控制器

# 查看更新策略
自动使用replicaset进行pod的管理。 
kubectl explain deploy.spec.strategy 
- strategy
    - type : Recreate/ RollingUpdate
    - maxSurge: 最多超过目标副本数
    - maxUnavailable: 最多有几个不可用
	两个值不能同时为0， 不然k8s无法控制进行滚动更新策略。

- revisionHistoryLimit: 保留多少个历史版本。
- template


code 
~~~ bash 
apiVersion: apps/v1
kind: Deployment

metadata: 
  name: myapp-deploy
  namespace: default

spec:
  replicas: 2
  selector: 
    matchLabels:
      app: myapp
      release: online
  template:
    metadata:
      labels: 
        app: myapp
        release: online
    spec:
      containers:
      - name: myapp-container
        image: mynginx:v1
        imagePullPolicy: IfNotPresent
        ports: 
        - name: httpd
          containerPort: 80
          
通过查找:
kubectl get pods 
kubectl get rs  //可以看一自行建立了replicaSet
kubectl get deploy 

每次修改的值保留在anotation中。
kubectl get pods -w  -l app=myapp 可以实时监控指定label的pod信息
~~~

指定更新的方式： 
1.  直接悠改文件，然后使用kubectl apply -f file.yaml重新生成一次
2.  kubectl set image
3.  kubectl patch
kubectl get rs -o wide 可以看到在更新的过程中， 可以看到有两个版本存在。 
kubectl rollout history deployment myapp-deploy
kubectl rollout history deployment myapp-deploy
直接使用patch命令进行打补丁，用于修改pod数量。 patch一定是补的是json格式的数值。
kubectl patch deploy myapp-deploy -p '{"spec":{"replicas":2}}'

修改deploy的滚动更新方式
kubectl patch deploy myapp-deploy -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1, "maxUnavailable":0}}}}'

直接set image
kubectl set image deployment myapp-deploy myapp=mynginx:v2 && kubectl rollout pause deployment myapp-deploy 

更新过程中，使用pause，将更新过程暂停。 这样可以实现金丝淮发布。 
kubectl rollout resume deploy myapp-deploy
kubectl rollout status 可以查看滚动更新的进程

先使用kubectl history 查看到对应的REVISION
kubectl rollout history deploy/myapp-deploy
kubectl rollout undo deploy/myapp-deploy --to-revision=3 回滚到对应的哪个版本


## deamonSet
kubectl explain ds
kubectl explain ds.spec

~~~ bash 
apiVersion: apps/v1
kind: DaemonSet

metadata: 
  name: demo-ds
  namespace: default

spec: 
  selector:
    matchLabels:
      appname: myapp

  template:
    metadata: 
      labels:
        appname: myapp
    spec:
      containers:
      - image: mynginx:v1
        name: demo-daemon-pod
        imagePullPolicy: IfNotPresent
~~~
kubectl get ds 

[root@k8smaster controllers]# kubectl get ds 
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
demo-ds   1         1         1       1            1           <none>          93s

--- 用于分隔不同的资源定义


kubectl set image daeamonsets filebeat-ds filebeat=ikubernetes/filebeat:5.5.6-alpine

~~~ bash 
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: redis-deployment
  namespace: default

spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template: 
    metadata:
        labels:
            app: redis
    spec: 
        containers:
        - name: rediscontainer
          image: redis:4.0-alpine
          imagePullPolicy: IfNotPresent
          ports:
          - name: redisport
            containerPort: 6379
    
---
apiVersion: apps/v1
kind: DaemonSet

metadata: 
  name: filebeat-ds
  namespace: default
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: ds-filebeat-container
        image: ikubernetes/filebeat:5.6.5-alpine
        env:
        - name: REDIS_HOST
          value: redis.default.svc.cluster.local
        - name: REDIS_LOG_LEVEL
          value: info 

~~~

pod中spec的hostNetwork，可以共享宿主机节点的网络名称空间。

