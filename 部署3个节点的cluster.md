
k8s-master 是 Master，k8s-node1 和 k8s-node2 是 Node。

1 前期准备
------

1.1 修改主机名及IP映射关系
==============
hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node1

1.2 修改>> /etc/hosts
==============
k8s-master 192.168.33.11
k8s-node1 192.168.33.12
k8s-node2 192.168.33.12

1.3 修改域名服务器 /etc/resolv.conf
==============
vim /etc/resolv.conf
nameserver 9.0.149.140
nameserver 9.0.146.50（改成你自己机器的域名服务器）

1.4 关闭并禁用防火墙。
==============
systemctl stop firewalld
systemctl disable firewalld

1.5 关闭SeLinux
==============
sed -i 's/enforcing/disabled/' /etc/selinux/config

1.6 禁用swap 内存交换
==============
swapoff -a
如果要永久禁止swap挂载，可以修改/etc/fstab，将与swap有关的配置注释，重启系统即可。

2 安装
-------------

2.1 安装 Docker （三台机器分安装）
==============
所有节点都需要安装 Docker。
apt-get update && apt-get install docker.io


2.2 安装 kubelet、kubeadm 和 kubectl（三台机器分安装）
==============
在所有节点上安装 kubelet、kubeadm 和 kubectl。
kubelet 运行在 Cluster 所有节点上，负责启动 Pod 和容器。
kubeadm 用于初始化 Cluster。
kubectl 是 Kubernetes 命令行工具。通过 kubectl 可以部署和管理应用，查看各种资源，创建、删除和更新各种组件。

apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl

如果安装指定版本：
apt-get install kubeadm=1.10.2-00 kubectl=1.10.2-00 kubelet=1.10.2-00

注意官方源 deb http://apt.kubernetes.io/ kubernetes-xenial main 下载不下来，改成aliyun


3 用 kubeadm 创建 Cluster
-----------------------

完整的官方文档可以参考 https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

3.1 初始化 Master
===================
在 Master 上执行如下命令：

kubeadm init --apiserver-advertise-address 192.168.33.11 --pod-network-cidr=10.244.0.0/16

--apiserver-advertise-address 指明用 Master 的哪个 interface 与 Cluster 的其他节点通信。如果 Master 有多个 interface，建议明确指定，如果不指定，kubeadm 会自动选择有默认网关的 interface。

--pod-network-cidr 指定 Pod 网络的范围。Kubernetes 支持多种网络方案，而且不同网络方案对 --pod-network-cidr 有自己的要求，这里设置为 10.244.0.0/16 是因为我们将使用 flannel 网络方案，必须设置成这个 CIDR。在后面的实践中我们会切换到其他网络方案，比如 Canal。

拉去镜像时出错：
=============
root@ubuntu1804:/home/vagrant# kubeadm init --apiserver-advertise-address 192.168.33.11 --pod-network-cidr=10.244.0.0/16
I0310 20:12:31.210363    4617 version.go:94] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: dial tcp: lookup dl.k8s.io on 127.0.0.53:53: server misbehaving
I0310 20:12:31.210444    4617 version.go:95] falling back to the local client version: v1.13.4
[init] Using Kubernetes version: v1.13.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-apiserver:v1.13.4: output: Error response from daemon: Get https://k8s.gcr.io/v2/: dial tcp: lookup k8s.gcr.io: Temporary failure in name resolution
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-controller-manager:v1.13.4: output: Error response from daemon: Get https://k8s.gcr.io/v2/: dial tcp: lookup k8s.gcr.io: Temporary failure in name resolution
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-scheduler:v1.13.4: output: Error response from daemon: Get https://k8s.gcr.io/v2/: dial tcp: lookup k8s.gcr.io: Temporary failure in name resolution
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-proxy:v1.13.4: output: Error response from daemon: Get https://k8s.gcr.io/v2/: dial tcp: lookup k8s.gcr.io: Temporary failure in name resolution
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/pause:3.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: dial tcp: lookup k8s.gcr.io: Temporary failure in name resolution
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/etcd:3.2.24: output: Error response from daemon: Get https://k8s.gcr.io/v2/: dial tcp: lookup k8s.gcr.io: Temporary failure in name resolution
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/coredns:1.2.6: output: Error response from daemon: Get https://k8s.gcr.io/v2/: dial tcp: lookup k8s.gcr.io: Temporary failure in name resolution
  
换国内的镜像源拉取：
==============
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.13.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.13.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.13.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.13.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.2.24
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.2.6

拉取成功后需tag成原来的镜像名
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.13.4 k8s.gcr.io/kube-apiserver:v1.13.4 
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.13.4 k8s.gcr.io/kube-controller-manager:v1.13.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.13.4 k8s.gcr.io/kube-scheduler:v1.13.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.13.4 k8s.gcr.io/kube-proxy:v1.13.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.2.6 k8s.gcr.io/coredns:1.2.6

注意：
==============
如果出现一下错误
Error response from daemon: Get https://registry.cn-hangzhou.aliyuncs.com/v2/: dial tcp: lookup registry.cn-hangzhou.aliyuncs.com: Temporary failure in name resolution

解决办法：
你用的是虚拟机，则需修改域名服务器
vim /etc/resolv.conf
nameserver 192.168.35.2（改成你自己机器的域名服务器） 


配置 kubectl
==============
kubectl 是管理 Kubernetes Cluster 的命令行工具，前面我们已经在所有的节点安装了 kubectl。Master 初始化完成后需要做一些配置工作，然后 kubectl 就能使用了。

依照 kubeadm init 输出的第步提示，推荐用 Linux 普通用户执行 kubectl（root 会有一些问题）。

我们为 ubuntu 用户配置 kubectl：

su - ubuntu
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
为了使用更便捷，启用 kubectl 命令的自动补全功能。

echo "source <(kubectl completion bash)" >> ~/.bashrc
这样 ubuntu 用户就可以使用 kubectl 了。


安装 Pod 网络
下节我们将安装 Pod 网络并添加 k8s-node1 和 k8s-node2，完成集群部署。
上节我们通过 kubeadm 在 k8s-master 上部署了 Kubernetes，本节安装 Pod 网络并添加 k8s-node1 和 k8s-node2，完成集群部署。


要让 Kubernetes Cluster 能够工作，必须安装 Pod 网络，否则 Pod 之间无法通信。

Kubernetes 支持多种网络方案，这里我们先使用 flannel，后面还会讨论 Canal。

执行如下命令部署 flannel：

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
593.png
 
3.2 添加 k8s-node1 和 k8s-node2
==============

在 k8s-node1 和 k8s-node2 上分别执行如下命令，将其注册到 Cluster 中：

kubeadm join --token d38a01.13653e584ccc1980 192.168.56.105:6443
这里的 --token 来自前面 'kubeadm init 输出的提示'，如果当时没有记录下来可以通过 kubeadm token list 查看。

===========
vagrant@ubuntu1804:~$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
podsecuritypolicy.extensions/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
vagrant@ubuntu1804:~$ 

kubeadm join 执行如下：
===========
root@ubuntu1804:/home/vagrant# kubeadm join 192.168.33.11:6443 --token 7guibg.3iwxlahq6idh15lc --discovery-token-ca-cert-hash sha256:8c7b4b6a3b8c212707333c4e14b2e2bb655b71fb35e055c3ec39e27b54a4b986
[preflight] Running pre-flight checks
[discovery] Trying to connect to API Server "192.168.33.11:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.33.11:6443"
[discovery] Requesting info from "https://192.168.33.11:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.33.11:6443"
[discovery] Successfully established connection with API Server "192.168.33.11:6443"
[join] Reading configuration from the cluster...
[join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "k8s-node2" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.


根据提示，我们可以通过 kubectl get nodes 查看节点的状态。
=========
vagrant@k8s-master:~$ kubectl get nodes
NAME         STATUS     ROLES    AGE    VERSION
k8s-master   Ready      master   21m    v1.13.4
k8s-node1    NotReady   <none>   2m7s   v1.13.4
k8s-node2    NotReady   <none>   6s     v1.13.4
vagrant@k8s-master:~$ 
vagrant@k8s-master:~$ 


目前所有节点都是 NotReady，这是因为每个节点都需要启动若干组件，这些组件都是在 Pod 中运行，需要首先从 google 下载镜像，我们可以通过如下命令查看 Pod 的状态：
kubectl get pod --all-namespaces
========
vagrant@k8s-master:~$ kubectl get pod --all-namespaces
NAMESPACE     NAME                                 READY   STATUS              RESTARTS   AGE
kube-system   coredns-86c58d9df4-6d9lb             1/1     Running             0          24m
kube-system   coredns-86c58d9df4-8jwxv             1/1     Running             0          24m
kube-system   etcd-k8s-master                      1/1     Running             0          24m
kube-system   kube-apiserver-k8s-master            1/1     Running             0          24m
kube-system   kube-controller-manager-k8s-master   1/1     Running             0          23m
kube-system   kube-flannel-ds-amd64-drftw          1/1     Running             0          13m
kube-system   kube-flannel-ds-amd64-q5fx5          0/1     Init:0/1            0          3m48s
kube-system   kube-flannel-ds-amd64-wzcqj          0/1     Init:0/1            0          5m49s
kube-system   kube-proxy-4kw5f                     0/1     ContainerCreating   0          3m48s
kube-system   kube-proxy-6gk5c                     0/1     ContainerCreating   0          5m49s
kube-system   kube-proxy-8vh7c                     1/1     Running             0          24m
kube-system   kube-scheduler-k8s-master            1/1     Running             0          24m

Pending、ContainerCreating、ImagePullBackOff 都表明 Pod 没有就绪，Running 才是就绪状态。我们可以通过 kubectl describe pod <Pod Name> 查看 Pod 具体情况，比如：
kubectl describe pod kube-proxy-4kw5f --namespace=kube-system
=========

  Normal   Scheduled               5m44s                 default-scheduler   Successfully assigned kube-system/kube-proxy-4kw5f to k8s-node2
  Warning  FailedCreatePodSandBox  99s (x12 over 5m37s)  kubelet, k8s-node2  Failed create pod sandbox: rpc error: code = Unknown desc = failed pulling image "k8s.gcr.io/pause:3.1": Error response from daemon: Get https://k8s.gcr.io/v2/: dial tcp: lookup k8s.gcr.io: Temporary failure in name resolution
  Warning  DNSConfigForming        40s (x15 over 5m43s)  kubelet, k8s-node2  Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 4.2.2.1 4.2.2.2 208.67.220.220

  说明 在k8s-node2 上需要的镜像文件没有获取到。

 执行拉去镜像操作
 换国内的镜像源拉取：
========
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.13.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.13.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.13.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.13.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.2.24
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.2.6

拉取成功后需tag成原来的镜像名
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.13.4 k8s.gcr.io/kube-apiserver:v1.13.4 
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.13.4 k8s.gcr.io/kube-controller-manager:v1.13.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.13.4 k8s.gcr.io/kube-scheduler:v1.13.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.13.4 k8s.gcr.io/kube-proxy:v1.13.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.2.6 k8s.gcr.io/coredns:1.2.6


systemctl start etcd
systemctl start docker
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kubelet
systemctl start kube-proxy



本章通过 kubeadm 部署了三节点的 Kubernetes 集群，后面章节我们都将在这个实验环境中学习 Kubernetes 的各项技术。







