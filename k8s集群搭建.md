kubeadm init \
> --pod-network-cidr=10.244.0.0/16 \
> --kubernetes-version v1.28.2 \
> --apiserver-advertise-address 10.67.108.215

### issues
#### 1
[/proc/sys/net/bridge/bridge-nf-call-iptables does not exist](https://blog.csdn.net/shida_csdn/article/details/99571884)

#### 2
[preflight] Some fatal errors occurred:
        [ERROR CRI]: container runtime is not running: output: time="2025-03-31T00:59:22+08:00" level=fatal msg="validate service connection: CRI v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
, error: exit status 1

##### 解决：
```bash
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
sudo kubeadm init
```

#### 3
```bash
[preflight] Some fatal errors occurred:
[ERROR ImagePull]: failed to pull image registry.k8s.io/kube-apiserver:v1.28.2: output: E0331 01:32:07.504696 1390199 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-apiserver:v1.28.2\": failed to resolve reference \"registry.k8s.io/kube-apiserver:v1.28.2\": failed to do request: Head \"https://registry.k8s.io/v2/kube-apiserver/manifests/v1.28.2\": dial tcp 34.96.108.209:443: i/o timeout" image="registry.k8s.io/kube-apiserver:v1.28.2"
time="2025-03-31T01:32:07+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-apiserver:v1.28.2\": failed to resolve reference \"registry.k8s.io/kube-apiserver:v1.28.2\": failed to do request: Head \"https://registry.k8s.io/v2/kube-apiserver/manifests/v1.28.2\": dial tcp 34.96.108.209:443: i/o timeout"
, error: exit status 1
```

*以上是在公司lab的机器上搭建集群，一直碰到镜像问题。换成家里的网络中搭建。*

//命令行配置使用阿里云的镜像
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version v1.28.2 --image-repository=registry.aliyuncs.com/google_containers
```

#### 4
初始化master节点的时候停在了这一步：[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s


- 接下来开始调试：

crictl pods命令返回：
```bash
WARN[0000] runtime connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead. 
E0331 10:58:48.058251   27587 remote_runtime.go:277] "ListPodSandbox with filter from runtime service failed" err="rpc error: code = Unavailable desc = connection error: desc = \"transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory\"" filter="&PodSandboxFilter{Id:,State:nil,LabelSelector:map[string]string{},}"
FATA[0000] listing pod sandboxes: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory"
```
从错误信息来看，**crictl 无法连接到容器运行时（container runtime）**，并且提示 **dockershim.sock 文件不存在**。这是因为**从 Kubernetes 1.24 开始，Docker 作为容器运行时已经被完全移除，Kubernetes 默认使用 containerd 或 CRI-O 作为容器运行时**。

- 检查容器运行时配置、运行情况
1. 首先，确认你的系统正在使用哪种容器运行时（containerd 或 CRI-O）
```bash
ps aux | grep -E 'containerd|crio'
```
*系统使用的是containerd，一下配置都以containerd为例*

2. 确保容器运行时已正确安装并运行
```bash
systemctl status containerd
```

3. 确保 containerd 的配置文件正确。默认情况下，containerd 的配置文件位于 /etc/containerd/config.toml
```bash
cat /etc/containerd/config.toml | grep systemd
```
- 现象：该文件不存在

- 原因：可能是因为 containerd 使用了默认配置，或者配置文件被生成到了其他位置

- 解决：

1. 生成默认的配置文件

```bash
containerd config default > /etc/containerd/config.toml
```

*检查 containerd 的实际配置路径,在某些情况下，containerd 可能将配置文件存储在其他位置。你可以通过以下方式查找实际的配置文件路径：查看 containerd 的 systemd 配置文件，确认是否有自定义的配置文件路径:*

```bash
cat /lib/systemd/system/containerd.service
```

*重点检查 ExecStart 参数，可能会看到类似以下的配置：*

```bash
ExecStart=/usr/bin/containerd --config /path/to/config.toml
```

2. 修改配置文件以启用 CRI

重点检查以下内容：

        - 确保 **[plugins."io.containerd.grpc.v1.cri"]** 部分存在且未被注释。

        - 确保 **sandbox_image** 配置正确（默认值为 k8s.gcr.io/pause:3.6）

        **把sandbox_image改成了阿里云的pause镜像**

保存后重启 containerd：sudo systemctl restart containerd


*到这里可以确认containerd正常运行*

- 指定 **crictl 的运行时端点**

如果使用的是 containerd：
```bash
crictl --runtime-endpoint unix:///run/containerd/containerd.sock pods
```
这个命令执行后什么结果都没返回 ———— 表明 crictl 无法连接到容器运行时


- 检查crictl是否能连接到运行时
```bash
crictl --runtime-endpoint unix:///run/containerd/containerd.sock info
```

- 现象：返回cni config load failed: no network config found in /etc/cni/net.d: cni plugin not initialized: failed to load cni config

- 原因：这表明 Kubernetes 的 CNI（Container Network Interface）网络配置缺失或未正确初始化。CNI 是 Kubernetes 中用于管理 Pod 网络的关键组件，缺少网络配置会导致 Pod 无法启动。

- 错误信息表明：

        - CNI 配置文件缺失：/etc/cni/net.d 目录下没有网络配置文件。

        - CNI 插件未初始化。

在 Kubernetes 中，CNI 配置文件通常位于 /etc/cni/net.d，并且需要安装一个网络插件（如 Flannel、Calico 等）来提供网络功能。

安装CNI网络插件的时候提示*The connection to the server localhost:8080 was refused - did you specify the right host or port?*

这表明 **kubectl 无法连接到 Kubernetes API 服务器**。通常，这种问题是由于 kubectl 的配置文件（~/.kube/config）丢失或配置错误导致的。

- 检查kubelet日志：
```bash
journalctl -u kubelet -f
```

- 现象： 命令提示：container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"

- 原因： 这表明 Kubernetes 的 CNI（Container Network Interface）网络插件未正确初始化。CNI 是 Kubernetes 中用于管理 Pod 网络的关键组件，缺少或未正确配置 CNI 插件会导致此问题。

错误信息表明：

- CNI 插件未初始化。

- 网络未准备好，导致 Kubernetes 无法正常运行 Pod。

Kubernetes 的网络插件通常需要一个配置文件（位于 /etc/cni/net.d）和一个 CNI 插件二进制文件（通常位于 /opt/cni/bin）。

检查 /etc/cni/net.d 目录
运行以下命令检查 /etc/cni/net.d 目录下是否有配置文件：

bash
ls -l /etc/cni/net.d

如果目录为空或没有配置文件：
你需要手动创建一个默认的 CNI 配置文件。

创建默认的 Flannel CNI 配置文件
运行以下命令创建一个默认的 Flannel 配置文件：

```bash
mkdir -p /etc/cni/net.d
cat <<EOF | sudo tee /etc/cni/net.d/10-flannel.conflist
{
"name": "cbr0",
"cniVersion": "0.3.1",
"plugins": [
{
"type": "flannel",
"delegate": {
        "isDefaultGateway": true
}
},
{
"type": "portmap",
"capabilities": {
        "portMappings": true
}
}
]
}
EOF
```
​检查 CNI 插件二进制文件
Kubernetes 的 CNI 插件需要二进制文件来执行网络操作。这些文件通常位于 /opt/cni/bin。

检查 /opt/cni/bin 目录
运行以下命令检查 /opt/cni/bin 目录下是否有 CNI 插件：

bash
ls -l /opt/cni/bin

*这个目录下是有内容的*

- 检查 kubectl 配置文件
kubectl 使用 ~/.kube/config 文件连接到 Kubernetes API 服务器。如果该文件丢失或配置错误，kubectl 将无法连接到 API 服务器。

运行以下命令检查 ~/.kube/config 是否存在：

```bash
ls -l ~/.kube/config
```
如果文件不存在，可能是因为 kubeadm init 未正确完成，或者你没有以正确的用户身份执行命令。
(本来就没有正确完成)

**重新生成 ~/.kube/config**

如果你是在 Master 节点上操作，可以重新生成 ~/.kube/config 文件。

在 Master 节点上运行以下命令：
```bash
sudo kubeadm init phase kubeconfig all
这将为当前用户生成正确的 ~/.kube/config 文件。
```

如果你以非 root 用户操作，需要手动复制配置文件：
```bash
sudo mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- ​验证 kubectl 是否能连接到 API 服务器
在修复配置文件后，验证 kubectl 是否能正常工作。

检查 Kubernetes 集群状态
运行以下命令检查集群节点状态：

```bash
kubectl get nodes
如果返回类似以下的输出，说明 kubectl 已正确连接到 API 服务器：
```
NAME       STATUS   ROLES           AGE   VERSION
master     Ready    control-plane   5m    v1.27.4
如果仍然报错，请继续排查。

- ​检查 API 服务器的地址和端口

kubectl 的配置文件中指定了 API 服务器的地址和端口。如果地址或端口不正确，kubectl 将无法连接。

检查 ~/.kube/config 文件

打开 ~/.kube/config 文件，检查 clusters[].cluster.server 配置项：

```bash
cat ~/.kube/config | grep server
```
输出应该类似于：

```bash
server: https://<master-ip>:6443
```
<master-ip> 是 Master 节点的 IP 地址。

端口 6443 是 Kubernetes API 服务器的默认端口。

如果地址不正确：确保 Master 节点的 IP 地址正确。

如果你在本地操作，可以使用 localhost 或 127.0.0.1 代替 <master-ip>。

*这边我改了一下master-ip，后面证明不该改这个*

- 检查防火墙设置 & 证书是否有效
关闭防火墙：ufw allow 6443/tcp


- 这个时候我突然福至心灵，觉得可以重新运行一下kubeadm init，于是我执行了一下kubeadm reset，很快就出现了如下结果：
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version v1.28.2 --image-repository=registry.aliyuncs.com/google_containers
[init] Using Kubernetes version: v1.28.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local metro-nuc13anki7] and IPs [10.96.0.1 192.168.10.10]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost metro-nuc13anki7] and IPs [192.168.10.10 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost metro-nuc13anki7] and IPs [192.168.10.10 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 4.503104 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node metro-nuc13anki7 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node metro-nuc13anki7 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: pw0ulp.iatuxyibfeu1dll9
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.10:6443 --token pw0ulp.iatuxyibfeu1dll9 \
    --discovery-token-ca-cert-hash sha256:26d89c34a272b5eeca2f29ad02d199c45659d6284dd8e581e5d2c110952d2732 
```
*不知道是哪个地方的修改生效了*


- 将运行时端点配置为默认值（可选）

编辑或创建 /etc/crictl.yaml 文件：

runtime-endpoint: unix:///run/containerd/containerd.sock

image-endpoint: unix:///run/containerd/containerd.sock

timeout: 10

debug: false

- 检查 kubelet 的配置：

kubelet 也需要正确配置以使用 containerd 或 CRI-O。确保 kubelet 的启动参数中指定了正确的容器运行时。

检查 kubelet 的配置文件（通常位于 /var/lib/kubelet/config.yaml 或通过 systemd 配置）。

如果使用的是 containerd：

确保 kubelet 的启动参数中包含：

containerRuntime: remote

containerRuntimeEndpoint: unix:///run/containerd/containerd.sock

#### 5
kubectl get nodes 返回错误：
```bash
Unable to connect to the server: tls: failed to verify certificate: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")
```

证书错误的话直接重新生成.kube/config文件

在 Master 节点上运行以下命令：
```bash
sudo kubeadm init phase kubeconfig all
这将为当前用户生成正确的 ~/.kube/config 文件。
```

如果你以非 root 用户操作，需要手动复制配置文件：
```bash
sudo mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 6
安装flannel插件后，kubectl get pod返回"The connection to the server localhost:8080 was refused - did you specify the right host or port?"

