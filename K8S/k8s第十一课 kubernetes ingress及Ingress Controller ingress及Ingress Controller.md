## k8s 第11课 kubernetes ingress及Ingress Controller ingress及Ingress Controller

service的工作模型
- userspace
- iptables
- ipvs

- 如何启动ipvs
    如果要启用ipvs , 可以修改改/etc/sysconfig/kubelet
    KUBELETE_EXTRA_ARGS="--fail-swap-on=false"
    KUBE_PROXY_MODE=ipvs

    启用内核模块和功能
    ip_vs, ip_vs_rr, ip_vs_sh，nf_contrack_ipv4

service的四种类型：
- clusterIP
- NodePort: 访问路径clientip --> nodeip:port --> clusterip:clusterport --> podip:pod-port
- ExternalName: 将集群外的服务引入集群内，绕过集群边界；FQDN格式的名称，要不是主机名，或是域名。是一个cname记录。 用于指向真正的地址
- loadBalancer: k8s如果部署在公有去，则可以利用公有去lbaas;

如果一个service中无clusterIP, 则变成了headless service . 此种模式下， serviceName --> podIP. 

pod , 控制器， service 是k8s中最基本的三个组件。 其他的是一些增强功能。 
4层调度无法谢载ssl会话， ssl 会话即贵且慢。 如果可以，那么可以在调度器上将ssl 谢载。 在不安全网络中使用ssl , 在安全网络中，使用正常的明文传输。 

使用一个具有七层功能的pod ， 用来转发和谢载ssl ，将后端连到pod上。 

### ingress controller
是独立运行的一组pod和程序，现在的解决方案有： 
- haproxy
- nginx
- Envoy
- Traefik
需要使用headless service 来给pod进行分组，并且实时更新pod的ip信息。 
ingress 可以注入到ingress controller中，当发生pod变化，会及时注入到ingress controller中。

ingress 流量结构
![](./ingress-controller.png)

4个核心附件： dns , heapster,ingress-controller , dashboard.


命令行创建namespace: kubectl create namespace dev
删除名称空间： kubectl delete namespace dev



live doc: https://kubernetes.github.io/ingress-nginx/deploy/
创建ingress: 

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml`

创建ingress

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml


查看NGINX的配置文件，可以留意到以下的关键信息：
~~~ yml
apiVersion: apps/v1
kind: Deployment

metadata: 
  name: nginx-deploy
  namespace: default

spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: nginx-myapp
        image: mynginx:v1
        imagePullPolicy: IfNotPresent
--- 
apiVersion: v1
kind: Service

metadata:
  name: nginx-deploy-service
  namespace: default

spec:
  selector:
    app: myapp
  clusterIP: None
  ports:
  - name: nginx-port
    port: 80
    protocol: TCP
    targetPort: 80

---
apiVersion: extensions/v1beta1
kind: Ingress

metadata:
  name: myapp-ingress
  namespace: default
  annotations: 
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: myapp.magedu.com
    http:
      paths:
      - backend:
          serviceName: nginx-deploy-service
          servicePort: 80

~~~


 kubectl exec -it  -n ingress-nginx nginx-ingress-controller-fd96b4f85-6szxx -- /bin/sh
 
进入后查看nginx.conf文件，可以重点留意以下几个信息：　
	## start server myapp.magedu.com
	server {
		server_name myapp.magedu.com ;


生成证书
openssl genrsa -out tls.key 2048
openssl req -new -x509 -key tls.key -out tls.crt --subj /C=CN/ST=BJ/L=Beijing/O=devops/CN=ttserver.com


kubectl create secret tls tomcat-ingress-secret --cert=tls.crt --key=tls.key
kubectl get secret
kubectl decribe secret

ingress-tls的配置：
~~~ bash 
apiVersion: extensions/v1beta1
kind: Ingress

metadata:
  name: tomcat-ingress-ssl
  namespace: default
  annotations: 
    kubernetes.io/ingress-class: "nginx"

spec:
  tls:
  - hosts:
    - tomcat.ttserver.com
    secretName:  tomcat-ingress-secret
  rules:
  - host: tomcat.ttserver.com
    http:
      paths:
      - backend:
          serviceName: tomcat-deploy-service
          servicePort: 8080


~~~

https 相关的nginx配置如下;
~~~ bash
	
		listen 443  ssl http2;
		
		# PEM sha: 9342896cfeac1340d108c0f27dbf2f9f37480e17
		ssl_certificate                         /etc/ingress-controller/ssl/default-fake-certificate.pem;
		ssl_certificate_key                     /etc/ingress-controller/ssl/default-fake-certificate.pem;

~~~

