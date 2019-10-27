## k8s第7课 kubernetes pod 控制器应用进阶

探针类型三种： 
ExecAction
TCPSocketAction
HTTPGET

liveniness 和 readiness是针对container进行探测的，不是针对pod进行探测。所以可以通过kubectl explain pod.spec.container 进行帮助查看。
- livenessProbe
	- kubecel explain pods.spec.conatiners.livenessProbe.exec 
	- command
	- 返回0表示状态健康，非0表示不健康
- redinessProbe
- lifecycle: 启动后，或是中止前钩子

kubectl get pods -w 实时查看pod 的状态

针对httpGet方式进行probe探测

~~~ bash 
apiVersion: v1
kind: Pod

metadata: 
  name: liveness-probe-httpget
  namespace: default

spec: 
  containers:
  - name: httpserver
    image: mynginx:v1
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80

    livenessProbe:
      httpGet:
        port: http
  restartPolicy: Never

~~~

readiness probe 必须做。 不然创建的pod会直接接入到service的服务中。 会将流程转发过来。 造成请求失效。 如果不做readiness ,很有可能在一段时间内，服务的请求是失败的。 
就续状态和存活状态都是不断进行探测的。不是只在某个时间段，比如创建容器时进行探测。 


~~~ bash 

apiVersion: v1
kind: Pod
metadata: 
  name: liveness-probe-httpget-pod

spec:
  containers:
  - name: liveness-probe-http
    image: mynginx:v1
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
    readinessProbe:
      httpGet:
        port: http
        path: /index.html
      initialDelaySeconds: 1
      periodSeconds: 3
~~~
readinessProbe探测，会在kubectl get pods 中的Ready状态，可以看到具体的信息。 

kubectl explain pods.spec.containers.lifecycle.preStop
lifecycle 中poststart命令，要晚于container中的command命令执行。 
命令执行的顺序
command --> post start



