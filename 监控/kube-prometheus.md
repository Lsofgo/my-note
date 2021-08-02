## 部署说明

使用 kube-prometheus 将 prometheus 直接部署到 k8s 集群中;

从三个维度监控k8s集群: 容器, 集群, 服务器



## kube-promethues部署

参考官方文档: https://prometheus-operator.dev/docs/prologue/quick-start/

**下载安装源码**

```
git clone https://github.com/prometheus-operator/kube-prometheus.git
```

 [kube-prometheus.zip](kube-prometheus.zip) 

**部署 kube-prometheus**

```shell
# Create the namespace and CRDs, and then wait for them to be availble before creating the remaining resources
cd kube-prometheus
kubectl create -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl create -f manifests/
```

安装过程中, k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.1.0 会无法下载, 可以把下面的镜像文件导入到每个node节点的docker中

 [kube-state-metrics_v2.1.0.tar](kube-state-metrics_v2.1.0.tar) 

```shell
docker load -i kube-state-metrics_v2.1.0.tar
docker tag cbcfc76083de k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.1.0
```

**把 prometheus-k8s alertmanager-main grafana 的Service 改成NodePort模式**

(略)







