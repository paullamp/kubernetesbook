# k8s 第十三课　kubernetes pv pvc configmap secret

pv 及pvc 刚才匹配，pvc才能申请成功。

特意设计了一种工作逻辑。storageclass , 一个标准资源类型。

NFS的存储空间，设计成一个分类；ceph的空间，可以划归成另一个类。 

或是由地理位置

或是由本地、云端存储，或是收费以及免费来区分。

存储设备必须具备restful风格的接口

