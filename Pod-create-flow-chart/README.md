## Kubernetes Pod 创建流程
Kubernetes Pod创建流程如果细节到每一个组件内的操作则会很复杂，这里重点讲解主要组件在创建Pod过程的调用流程以及大致作用。如果想了解每个组件内具体操作的童鞋我也会提供一张详细的流程图。
### 1. Kubernetes主要组件介绍
如果想更好的了解接下去的流程，一定想要理解Kubernetes中核心组件及其功能。

 - **ETCD**：分布式高性能键值数据库,存储整个集群的所有元数据
 - **ApiServer**: API服务器,集群资源访问控制入口,提供restAPI及安全访问控制
 - **Scheduler**：调度器,负责把业务容器调度到最合适的Node节点
 - **Controller Manager**：控制器管理,确保集群资源按照期望的方式运行。主要包含以下Controller：
Replication Controller
Node controller
ResourceQuota Controller
Namespace Controller
ServiceAccount Controller
Token Controller
Service Controller
Endpoints Controller
 - **kubelet**：运行在每个节点上的主要的“节点代理”
**1.**pod 管理****：kubelet 定期从所监听的数据源获取节点上 pod/container 的期望状态（运行什么容器、运行的副本数量、网络或者存储如何配置等等），并调用对应的容器平台接口达到这个状态。
**2.**容器健康检查****：kubelet 创建了容器之后还要查看容器是否正常运行，如果容器运行出错，就要根据 pod 设置的重启策略进行处理.
**3.容器监控**：kubelet 会监控所在节点的资源使用情况，并定时向 master 报告，资源使用数据都是通过 cAdvisor 获取的。知道整个集群所有节点的资源情况，对于 pod 的调度和正常运行至关重要
 - **CRI(Container Runtime Interface)**: Kubernetes提供的一个插件接口，它使 kubelet能够与各种容器运行时进行通信。容器运行时通常是Containerd，Docker等
 - **kube-proxy**：维护节点中的iptables或者ipvs规则

这些组件中kube-proxy的主要作用是集群内的Service流量转发以及负载均衡，并不直接参与Pod创建。其余主要组件都深度参与Pod创建。
### 2. Kubernetes List-Watch模式

> 理解这种工作模式对理解接下来的流程图会很有帮助。Kubernetes中controller-manager，scheduler，kubelet都采取List-Watch工作模式与kube-apiserver通信。

在 Kubernetes 中，List-Watch 模式是一种常见的模式，用于使各种控制器、调度器、kubelet 等组件与 API server 通信，以便实时监控资源的变化。这种模式结合了列表 (List) 和观察 (Watch) 两种操作，允许组件在 Kubernetes API server 中的资源发生变化时获得通知。

这是 List-Watch 模式的基本工作方式：
**List:** 当组件首次启动或重新同步时，它会调用 List 操作，从 API server 获取其关心的资源当前状态的完整列表。例如，ReplicaSet 控制器可能会获取所有的 ReplicaSet 资源。

**Watch:** 列表操作完成后，组件开始 Watch 操作。这意味着它们会建立一个长连接，等待 API server 发送有关资源更改的通知（例如新建、更新或删除资源）。这样，组件可以实时获知资源状态的变化，并根据需要做出反应。
### 3. Kubernetes Pod创建流程图

![在这里插入图片描述](https://img-blog.csdnimg.cn/190ad72d0c9d454480a7329a8347b97a.png)
1. 包含图中第1步，由用户通过kubectl调用kube-apiserver接口创建一个ReplicaSet资源（也可以是其他资源，这里假设为ReplicaSet）。
2. 包含图中第2步，kube-apiserver将要新创建的ReplicaSet资源信息(yaml文件表示的信息)写入etcd。
3. 包含图中第0、3、4步，controller-manager通过List-Watch模式从kube-apiserver处获取所有其控制资源的数据，当然会包含新创建的ReplicaSet资源的数据。controller-manager收到新建ReplicaSet资源的数据后，对比当前ReplicaSet资源已有的Pod副本数，以及期望的Pod副本数。
4. 包含图中第5、6步，由于ReplicaSet资源已有的Pod副本数，以及期望的Pod副本数之间肯定存在差异，controller-manager会将创建Pod的资源信息(yaml文件表示的信息)递给kube-apiserver，kube-apiserver而后将新建Pod资源信息(yaml文件表示的信息)写入etcd。
5. 包含图中0、7、8步，scheduler通过List-Watch模式从kube-apiserver处获取所有其需要的资源的数据，当然会包含需要新建一个Pod的数据。大致的方式是查看是否所有Pod都已经分了节点资源，此时就会发现有Pod并未分配到节点。
6. 包含图中第9、10步，scheduler通过调度算法选择一个节点分配给新创建的Pod，scheduler会通过kube-apiserver将数据最终写入到etcd中。
7. 包含图中0、11、12步，kubelet通过List-Watch模式从kube-apiserver处获取所有其需要的资源的数据，当然会包含需要在当前节点新启动一个Pod的信息。大致的方式是得到当前节点所有需要运行的Pod信息，发现有一个Pod还未处于运行状态。
8. 上面图中并未画出以下步骤，kubelet会通过CRI(Container Runtime Interface)通知到当前节点上的容器运行工具（如：Docker）需要创建一个新的Pod。完成创建后容器运行工具也会通过CRI(Container Runtime Interface)告知kubelet当前Pod状态。进一步kubelet也会将新创建的容器状态告知kube-apiserver而后更新etcd中有关当前Pod的资源信息。
### 4. 更加详细的流程图
实际Kubernetes新建一个资源的过程十分复杂，如果想了解更多可以尝试理解下面的流程图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/1ca2a684483242bc9325717e5871be74.png)
