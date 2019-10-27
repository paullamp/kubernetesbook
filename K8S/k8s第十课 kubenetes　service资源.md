## k8s第10课 kubernetes service资源
k8s集群中有三种网络
- node network: 节点的实际网络
- pod network :pod网段，实际在pod中可以查看到
- cluster network :虚拟网络， 只存在规则表中。
- apiservice < ---watch---> kube-proxy 
- 三种转发模型
	- userspace（1.1-)
	- iptables(1.10-)
	- ipvs(1.11+)
	- 转换是动态的， 实时的

service 类型： 
 ExternalName, ClusterIP, NodePort, and LoadBalancer, 默认是ClusterIP

ExternalName: 用于实现本地pod连接外部服务。即pod节点是在集群外部。 使用的pod能像使用集群内部的服务一样，使用外部的服务。 


ports中有三个端口的定义： 
- targetPort: 容器内运行的端口
- port:  svc服务的地址
- nodePort: 当类型为nodePort时，可以在集群外部通过node network访问到的端口

~~~ bash 
apiVersion: v1
kind: Service

metadata: 
  name: redis-svc

spec: 
  selector:
    app: redis
  type: NodePort
  ports: 
  - name: redis-service
    port: 6379
    targetPort: 6379
~~~

kubectl describe svc redis-svc

endpoint就是ip:端口
资源记录格式：
svc_name.NS_NAME.DOMAIN.LTD 默认后缀是svc.cluster.local.

如果想创建nodePort型的资源，则只要将类型改为nodePort即可。 nodeport是动态分配的， 从30000到32767随机分配。
测试nodeport的访问效果
while true; do curl http://172.20.0.66:30080/hostname.html;sleep 1;done


支持sessionAffinity, 即会话粘性。
kubectl patch svc myapp -p '{"spec":{"sessionAffinity":"ClientIP"}}'
kubectl patch svc myapp -p '{"spec":{"sessionAffinity":"None"}}' 默认值是none

Headless service
正常的情况是clientip ===> svcname: clusterip:port ==> pod ip:targetPort
当设置成无头service时， 需要将ClusterIP: "", 设置为空。此时，svcname直接连解析到pod ip上。使用dns的轮循方式进行后端节点的调度。

code
~~~ bash 
[root@k8smaster controllers]# cat demo-svc.yaml 
apiVersion: v1
kind: Service

metadata: 
  name: redis-svc

spec: 
  selector:
    app: redis
  clusterIP: None
  ports: 
  - name: redis-service
    port: 6379
    targetPort: 6379

[root@k8smaster controllers]# kubectl get svc 
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP    82d
redis-svc    ClusterIP   None         <none>        6379/TCP   12s
[root@k8smaster controllers]# 
~~~

使用ingress 来进行七层调度
nginx
traffic
invoic