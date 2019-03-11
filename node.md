Node 节点
===

Node 是 Pod 运行的地方，Kubernetes 支持 Docker、rkt 等容器 Runtime。 <br>
Node上运行的 Kubernetes 组件有:
* kubelet、
* kube-proxy 和 Pod 网络（例如 flannel）

kubelet
---
kubelet 是 Node 的 agent，当 Scheduler 确定在某个 Node 上运行 Pod 后，会将 Pod 的具体配置信息（image、volume 等）发送给该节点的 kubelet，kubelet 根据这些信息创建和运行容器，并向 Master 报告运行状态。

kube-proxy
---
service 在逻辑上代表了后端的多个 Pod，外界通过 service 访问 Pod。service 接收到的请求是如何转发到 Pod 的呢？这就是 kube-proxy 要完成的工作。

每个 Node 都会运行 kube-proxy 服务，它负责将访问 service 的 TCP/UPD 数据流转发到后端的容器。如果有多个副本，kube-proxy 会实现负载均衡。

Pod 网络
---
Pod 要能够相互通信，Kubernetes Cluster 必须部署 Pod 网络，flannel 是其中一个可选方案。

<br>
参考：<br>
https://www.cnblogs.com/CloudMan6/p/8308334.html

