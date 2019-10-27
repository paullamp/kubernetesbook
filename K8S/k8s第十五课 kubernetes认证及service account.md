# k8s 第十五课　kubernetes 认证及service account



k8s是我们运行无状态程序的重要平台。托管了大量的无状态应用。

因此，针对k8s内的服务，都应该进行授权访问。 

k8s的模型设计是足够安全的。

apiserver是针对内部的资源对象管理，操作apiserver前，必须先进行安全认证。

操作资源需要进行以下几步：

1. 认证
2. 授权检查
3. 准入控制，级连其他资源或是相应的环境

高度模块化设计，以上功能都是以插件的形式进行管理。

认证方式：



1. token: restful传递的密钥一般称作为token.承载令牌。客户端需要带着令牌去服务器端进行认证。只需要交换以确认是否具有预共享密钥即可。 

2. ssl 认证：通过公私钥的方式进行认证，通过证书形式以认证各个功能模块
3. 只需要经过一个认证插件成功，即表示成功。不需要进行串行检查。

授权模式，也支持多种插件。 k8s从1.6开始支持RBAC, rbac是许可授权的，而非拒绝授权。rbac可以定制的非常灵活。

准入控制是指在在授权结束后的其他安全检查机制。

用户请求client -> apiserver时，需要提供以下的信息：

- username , uid

- group

- extra

- apiresource 
  - /apis/apps/v1/namespaces/default/deployments/myapp-deploy  CURD增删改查
- namespace名称空间级别的资源

alpha 内测， beta公测， stable稳定版

复制的.kube/admin.conf的认证信息分析。

kubectl proxy --port=8080

curl http://127.0.0.1:8080/apis/v1/namespaces 核心群组的路径

curl http://127.0.0.1:8080/api/v1/namespaces 通过此方法，可以直接绕过https请求，直接访问http形式的资源。

00：23