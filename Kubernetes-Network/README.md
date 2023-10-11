## 综述
本文的主要目的是解释Kubernetes中有网络相关的概念区分，便于读者更好的理解Kubernetes网络构成。主要介绍kube-prox, CNI, 服务发现（DNS）三者之间的区别及作用。会侧重每个概念在Kubernetes集群中起到的作用，但是为了方便理解不会涉及概念的具体实现方式。
### 1. CNI
CNI（Container Network Interface）是一个规范，定义了如何创建和配置容器网络。CNI 插件用于实现这些规范，以便 Kubernetes 集群中的容器能够互相通信和访问外部网络。CNI 的作用如下：

1. **容器通信**: CNI 插件负责创建和管理容器之间的网络连接。它确保在同一节点上的不同容器可以相互通信，无论这些容器属于同一个 Pod 还是不同的 Pod。

2. **跨节点通信**: CNI 插件使得不同节点上的容器也可以互相通信。这是 Kubernetes 中跨节点 Pod 通信的基础，它通过网络层路由和隧道技术来实现。

3. **IP 地址分配**: CNI 插件管理 IP 地址的分配和回收。每个容器都需要一个唯一的 IP 地址，CNI 负责为容器分配合适的 IP 地址，并在容器停止时释放这些地址，以便重新使用。

4. **外部网络访问**: CNI 插件还负责配置容器以访问外部网络，包括互联网或企业内部网络。

总的来说，Kubernetes 是有多个节点（Node）组成，不同的节点可能还处于不同的网段，集群网络网络的作用就是将运行在这些节点的 Pod 可以看成在同一个局域网，不同的 Pod 可以无障碍的相互通信。常见的 CNI 插件有例如 Calico, Flannel, Weave 等。
### 2. kube-proxy
kube-proxy 运行在每个节点上为 Kubernetes 服务提供网络代理功能。以下是 kube-proxy的主要作用：

1. **服务访问**: 当你在 Kubernetes 中创建一个服务，它会有一个与之关联的虚拟 IP（称为 ClusterIP）。kube-proxy 负责监听这些服务的虚拟 IP，以及相关的端口，当有流量到达这些 IP/端口时，kube-proxy 会将其转发到与该服务关联的 Pod。

2. **负载均衡**: 当一个服务有多个 Pod 副本时，kube-proxy 会自动地负载均衡进入服务的流量，确保每个 Pod 得到均衡的请求量。

3. **维护 Endpoints**: Kubernetes 的 Endpoints 是实际运行的 Pod IP 列表，这些 Pod 背后的服务。当 Pod 进行伸缩、出现故障或部署更新时，kube-proxy 负责更新这些 Endpoints，确保流量只被转发到健康的、实际运行的 Pod。

4. **NodePort 和 LoadBalancer**: kube-proxy 还支持 Kubernetes 的 NodePort 和 LoadBalancer 服务类型，它们允许外部流量进入集群。对于 NodePort，kube-proxy 在所有节点上的指定端口上监听外部流量，并转发到服务的 Pod。对于 LoadBalancer，kube-proxy 与云提供商的负载均衡器协作，将外部流量引导到集群中。

总的来说 kube-proxy 更加侧重集群内（ClusterIP）外（NodePort，LoadBalancer）流量的转发及负载均衡。将集群内 Pod 的流量通过 IP 地址转发到一个集群内 IP 地址（ClusterIP方式）或集群节点上的一个特殊端口（NodePort方式），也可以是一个特定的集群外 IP 地址（LoadBalancer方式）。kube-proxy 最主流的实现方式为iptables或ipvs。
### 3. 服务发现（DNS）
Kubernetes 中，DNS 它为服务发现和名称解析提供了核心功能。它使得 Pod 和服务能够通过名称而不是 IP 地址进行通信，从而提供了灵活性、动态性和解耦。以下是 Kubernetes 中 DNS 的主要作用：

1. **服务发现**: 当 Pod 需要与其他服务通信时，它可以使用服务的 DNS 名称来进行连接，而不需要知道该服务背后的任何具体 IP 地址。当你创建 Kubernetes 中的 Service，一个与之关联的 DNS 名称会自动被创建，这样其他 Pod 就可以使用这个名称来访问该服务。

2. **抽象和解耦**: 使用 DNS 名称允许应用程序在 Kubernetes 内部通过逻辑名称进行通信，而不是通过硬编码的 IP 地址。这提供了一种解耦方式，允许服务的 Pod 可以动态地更改、伸缩或迁移，而不影响到消费者。

3. **稳定的访问点**: Kubernetes 服务的 DNS 名称提供了一个稳定的访问点，即使背后的 Pod 地址经常更改。

4. **自定义 DNS 记录**: 在某些场景下，例如使用 StatefulSet，DNS 还可以为每个 Pod 提供特定的、可预测的 DNS 名称。

其实理解服务发现（DNS）的作用和我们访问网站时需要用DNS解析的原理一致，一个域名背后可能对应这多 IP 地址，这些 IP 地址可能随着时间及我们所处不同的地理位置发生变化。当拥有一个DNS域名解析时用户就不必关注域名之后的 IP 地址了。
Kubernetes 中，默认的 DNS 服务提供者是 CoreDNS (曾经是 Kube-DNS)。
