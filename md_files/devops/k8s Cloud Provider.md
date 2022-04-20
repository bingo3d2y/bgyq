## k8s Cloud Provider

要不读读他的代码？

https://github.com/kubernetes/cloud-provider



为了更好地让 Kubernetes 在公有云平台上运行，并且提供容器云服务，云厂商需要实现自己的 Cloud Provider，即实现：cloudprovider.Interface

> https://github.com/kubernetes/cloud-provider/blob/618368f33fdd39f642ddb1c003c3cd1350da9727/cloud.go#L17

它是 Kubernetes 中开放给云厂商的通用接口，便于 Kubernetes 自动管理和利用云服务商提供的资源，这些资源包括虚拟机资源、负载均衡服务、弹性公网 IP、存储服务等。 

### k8s + Openstack

kubelet 去调nova创建虚拟机啊 6

The `kubelet` service accesses `nova` to obtain the `nova` instance name. The kubelet node name will be set to the instance name. It also accesses `cinder` to mount `PersistentVolumes` that are requested by a pod.

The `kube-apiserver` service accesses `nova` to limit admission to the Kubernetes cluster. The service only allows the `cloudprovider`-enabled Kubernetes nodes to register themselves into the cluster.

The `openstack-cloud-controller-manager` service accesses `cinder` and `neutron` to create `PersistentVolumes` (using `cinder`) and `LoadBalancers` (using `neutron-lbaas`).

Below is a diagram of the components involved and how they interact.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/os-cloud-provider-k8s.png)

#### metadata.go

代码逻辑，从openstack cloud-init挂载的metadata中读取元数据信息

1. 定义从哪读取数据

   configdrive：https://github.com/kubernetes/kubernetes/blob/c1e69551be1a72f0f8db6778f20658199d3a686d/staging/src/k8s.io/legacy-cloud-providers/openstack/metadata.go#L108

   即："/dev/disk/by-label/config-2" 

   metadataservice：https://github.com/kubernetes/kubernetes/blob/c1e69551be1a72f0f8db6778f20658199d3a686d/staging/src/k8s.io/legacy-cloud-providers/openstack/metadata.go#L98:6

   即169.254.169.254

2. 遍历可以读取metadata的方式

   https://github.com/kubernetes/kubernetes/blob/c1e69551be1a72f0f8db6778f20658199d3a686d/staging/src/k8s.io/legacy-cloud-providers/openstack/metadata.go#L179

当虚拟机无法从configdrive和metadataservice获取数据时就好报错。

#### kubelet配置

```bash

$ cat  /etc/kubernetes/kubelet.env
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=2"
KUBELET_ADDRESS="--node-ip=10.200.180.117"
KUBELET_HOSTNAME="--hostname-override=bbbbbbbbbbbb-17057"

KUBELET_ARGS="--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \
--config=/etc/kubernetes/kubelet-config.yaml \
--kubeconfig=/etc/kubernetes/kubelet.conf \
--pod-infra-container-image=os-harbor-svc.default.svc.cloudos:11443/helm/k8s.gcr.io/pause:3.1 \
--runtime-cgroups=/systemd/system.slice \
   "
KUBELET_NETWORK_PLUGIN="--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
## openstack 租户账户信息
KUBELET_CLOUDPROVIDER="--cloud-provider=openstack --cloud-config=/etc/kubernetes/cloud_config"
```

end

### 批量情况k8s node pod

1、 先清理三个节点的内存缓存 

sync 下刷内存

echo 3 > /proc/sys/vm/drop_caches 释放缓存



2、 清理状态为 extied和create的容器  三个节点都清理下 

docker rm $(docker ps -a |grep "Exited\|Create" |awk '{print $1}')

 docker rmi $(docker images -q) -f

3、 再强制删除异常的pod

kubectl delete pod 容器名称 --force --grace-period=0

 

如下面清理unknown容器
