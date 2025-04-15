## OVN
- 网络模式：
    - VXLAN：可以跨物理网络，会增加封装开销
    - Underlay:性能更高，要求节点在同一个二层网络

- 节点网络配置：
    - 网关和路由规则

## Flannel
[Git](https://github.com/flannel-io/flannel)

Flannel 在每个主机上运行一个名为 flanneld 的小型二进制代理，并负责从更大的预配置地址空间中为每个主机分配子网。

Flannel 直接使用 Kubernetes API 或 etcd 来存储网络配置、分配的子网和任何辅助数据（例如主机的公共 IP）。数据包使用几种后端机制之一转发，包括 VXLAN 和各种云集成。

### 网络细节

Kubernetes 等平台假设每个容器（pod）在集群内都有一个唯一的可路由 IP。这种模型的优点是它消除了共享单个主机 IP 带来的端口映射复杂性。

Flannel 负责在集群中的多个节点之间提供第 3 层 IPv4 网络。Flannel 不控制容器如何联网到主机，只控制主机之间流量的传输方式。但是，flannel 确实为 Kubernetes 提供了一个 CNI 插件，并提供了与 Docker 集成的指导。

Flannel 专注于网络。对于网络策略，可以使用 Calico 等其他项目。

- 网络模式：
    - Overlay的VXLAN或UDP封装

- [backend](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)


