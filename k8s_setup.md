
k8s-master 是 Master，k8s-node1 和 k8s-node2 是 Node。

安装 Docker
------------------
所有节点都需要安装 Docker。

apt-get update && apt-get install docker.io


安装 kubelet、kubeadm 和 kubectl
------------------
在所有节点上安装 kubelet、kubeadm 和 kubectl。

kubelet 运行在 Cluster 所有节点上，负责启动 Pod 和容器。

kubeadm 用于初始化 Cluster。

kubectl 是 Kubernetes 命令行工具。通过 kubectl 可以部署和管理应用，查看各种资源，创建、删除和更新各种组件。

apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get updateapt-get install -y kubelet kubeadm kubectl
