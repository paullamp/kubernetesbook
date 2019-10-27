## k8s第四课 kubernetes应用快速入门

### 演示安装，并且一些基本命令及原理
kubectl 使令解析。详解每一个命令以及命令的使用场景
k8s的最小高度单元是pod

小实验：创建一个nginx pod ,并且检查pod内的网络，使用nginx的内容可在在外部被访问到


#### kubectl 常用功能演示

- kubectl run 
	- 运行的是一个deploy or job , cronjob类型 
	- create and run a particular image, possibly replicated.
	- create a deployment or job to manage the created container
	- kubectl run nginx --image=nginx:1.16.0-alpine
	- kubectl run nginx --image=nginx:1.16.0-alpine --dry-run=true
	- kubectl run nginx --image=nginx:1.16.0-alpine --replicas=5
	- kubectl run nginx --image=nginx:1.16.0-alpine --restart=Never
	- kubectl run nginx --image=nginx:1.16.0-alpine -- /bin/sh
	- pod所属的网络为cni0桥，不再属于docker0桥
	- pod的客户端有两类，一类是集群外部的pod，一类是其他pod 
	- 直接启动一个buxybox用于项目调试使用
	- 如果增加了--restart=Never选项，那么只会以单pod的形式运行，而非以deploy形式运行
	- kubectl run nginx-client --image=busybox -it --restart=Never
	- wget -q -O - nginx-deploy:8888
	- kubectl run nginx-client --image=nginx:1.16.0-alpine --restart=Never -it  -- /bin/sh

- kubectl exec 
	- kubectl exec pod nginx-5d67669d8f-kgrzv ip addr show  
	- 集存的dns搜索域:default.svc.cluster.local , svc.cluster.local,cluster.local 

- kubectl get
	- kubectl get pods -o wide
	- kubectl get svc -o wide 
	- kubectl get services -o wide
	- kubectl get svc -n kube-system -o wide
	- kubectl get pods --show-labels
	- kubectl get deploy -w 
- kubectl delete 
	- kubectl delete pods nginx-5d67669d8f-kgrzv 
	- kubectl delete deploy nginx

- kubectl expose
	- kubectl expose --help
	- kubectl expose deployment nginx-deploy --port=8888 --target-port=80 --dry-run=true 

- kubectl describe
	- kubectl describe svc nginx 

- kubectl edit 
	- kubectl edit svc nginx-deploy 

- kubectl scale 
	- kubectl scale --replicas=5 deployment myapp
- kubectl set image --help
	- kubectl set image deploymnet myapp myapp=ikubernetes/myapp:v2
- kubectl rollout
	- kubectl rollout status deployment myapp
	- kubectl rollout undo deployment myapp

小实验：　
演示pod的动态变动功能,可能通过scale 配合pod变动,强调service的稳定访问端点功能
演示并解说service的功能
制作自己的ikubenetes/myapp:v1/v2
解析iptables 规则，详细梳理流量走向