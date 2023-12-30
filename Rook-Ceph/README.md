## Kubernetes集群部署Rook Ceph部署Ceph集群
### 1. Rook Ceph介绍
Rook Ceph是Rook项目中的一个存储方案，专门针对Ceph存储系统进行了优化和封装。Ceph是一个高度可扩展的分布式存储系统，提供了对象存储、块存储和文件系统的功能，广泛应用于提供大规模存储解决方案。将Ceph与Rook结合，目的是利用Rook的云原生存储编排能力来简化Ceph在Kubernetes环境中的部署和管理。
[Rook Ceph官网](https://rook.github.io/docs/rook/latest-release/Getting-Started/intro/)
[Rook Ceph项目代码仓库](https://github.com/rook/rook)

Rook Ceph的主要特点包括：

1. **云原生集成**：Rook Ceph紧密整合了Kubernetes的特性，使Ceph存储系统能够以Kubernetes原生的方式进行部署和管理。这包括使用Kubernetes自定义资源定义（CRDs）来管理Ceph集群的配置和状态。

2. **自动化管理**：Rook自动化了Ceph的部署、配置、扩展、升级和故障处理。这意味着Rook会帮助自动管理Ceph集群的复杂性，用户可以更专注于使用存储而不是管理存储。

3. **灵活的存储选项**：通过Rook Ceph，用户可以在Kubernetes上部署和使用Ceph的对象存储、块存储和文件系统，满足不同类型的存储需求。

4. **扩展性和高可用性**：Ceph本身就是为了提供高度的扩展性和可靠性设计的。结合Rook，它可以在Kubernetes环境中自动扩展和恢复，提供持续可用的存储服务。

5. **简化操作**：Rook Ceph提供了简化的安装过程，通过几个简单的命令就可以在Kubernetes集群上部署一个完整的Ceph存储系统。

6. **社区支持**：Rook是一个活跃的开源项目，拥有广泛的社区支持，Rook Ceph作为其中的一部分，也继承了这种活跃和开放的社区特性。

使用Rook Ceph，用户可以更容易地在Kubernetes上部署和管理Ceph集群，享受到Ceph强大的存储功能，同时又不需要深入了解和管理其复杂的配置和维护任务。这使得Rook Ceph成为希望在Kubernetes环境中使用高可靠、可扩展存储解决方案的用户的一个理想选择。
### 2. 对比一下几种主流的Ceph安装方式
Rook Ceph, ceph-deploy, ceph-ansible, 和 cephadm 是部署和管理Ceph集群的不同工具和方法，每种方式都有其特点和适用场景。

#### 2.1 Rook Ceph

**特点**:
- **云原生集成**：特别为Kubernetes环境设计，利用Rook自动化管理Ceph集群。
- **操作简便**：通过Kubernetes CRDs进行集群管理，用户无需直接与Ceph集群交互。
- **自动化管理**：自动处理部署、扩展、升级和故障恢复等。
- **官方支持**：虽然不是Ceph官方直接支持的工具，但作为CNCF项目，在云原生生态中得到广泛认可。

**适用场景**:
- Kubernetes环境下，希望通过声明式API管理Ceph存储。
- 寻求简化Ceph集群管理和自动化操作的环境。

#### 2.2 ceph-deploy

**特点**:
- **简单直接**：命令行工具，适合快速部署小规模Ceph集群。
- **易于上手**：对于新手友好，不需要复杂的配置。
- **官方支持**：Ceph早期版本中推荐的部署工具和集群管理工具。它能够简化部署过程，自动配置和安装Ceph节点，并提供一些管理操作的便捷功能。当前ceph-deploy已经不再被维护，而且它还没有在比Ceph 14 (Nautilus)更新的Ceph 版本上进行测试，不支持 RHEL8、CentOS 8或更新版本的操作系统。

**适用场景**:
- 小型或测试环境，快速部署Ceph集群。
- 不需要复杂或高度自定义的部署。

#### 2.3 ceph-ansible

**特点**:
- **灵活性高**：使用Ansible Playbooks，提供了更丰富的配置选项和自定义能力。
- **大规模部署**：适合大规模和生产环境的Ceph部署。
- **社区支持**：Ansible社区提供丰富的模块和角色支持。

**适用场景**:
- 需要定制化配置和管理较大规模Ceph集群的环境。
- 已经使用Ansible作为自动化工具的团队。

#### 2.4 cephadm

**特点**:
- **官方工具**：Ceph官方推荐的部署工具，从Ceph 15 Octopus版本开始引入。
- **容器化部署**：使用容器化方式部署Ceph服务。
- **简化操作**：提供了简化的命令行接口，易于维护和升级。

**适用场景**:
- 寻求官方支持和维护的部署方法。
- 希望通过容器化技术部署和管理Ceph集群。
- 需要易于升级和维护的环境。
### 3. 基本要求介绍
本文档使用的Rook是v1.13.1，Ceph(Reef) 18.2.1。(最新版本，暂时不推荐生产环境使用)

当前Rook支持Kubernetes v1.22或更高版本。

Rook发布包支持的架构包括amd64/x86_64和arm64。

在部署Rook之前，要检查Kubernetes集群是否准备好使用Rook，请查看以下先决条件:

要配置Ceph存储集群，至少需要以下其中一种本地存储：
 - 原始设备(Raw devices)（无分区或格式化文件系统）
 - 原始分区(Raw partitions)（无格式化文件系统）
 - LVM逻辑卷（无格式化文件系统）
 - 以块模式(block mode)在存储类(storage class)中提供的持久卷

Kubernetes集群信息：
机器名     | k8s版本 | 系统版本 | 系统内核 | cpu架构
-------- | -----  | -----   | -------  | -------
yiqi-rancher-master1  | v1.26.8  | CentOS Linux release 7.9 |   5.4.242-1.el7 |   x86_64
yiqi-rancher-master2  | v1.26.8  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86_64
yiqi-rancher-master3  | v1.26.8  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86_64
yiqi-rancher-worker1  | v1.26.8  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86_64
yiqi-rancher-worker2  | v1.26.8  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86_64
yiqi-rancher-worker3  | v1.26.8  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86_64
yiqi-rancher-worker4  | v1.26.8  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86_64
yiqi-rancher-worker5  | v1.26.8  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86_64

每个Work节点都有vdb，vdc，vdd三个原始设备（Raw devices）：
```bash
lsblk -f
NAME            FSTYPE      LABEL UUID                                   MOUNTPOINT
vdd                                                                      
vdb                                                                      
vdc                                                                      
vda                                                                      
├─vda2          LVM2_member       rgicKt-MO6U-rHDf-W7YC-VbCS-Rcl8-XU1vQE 
│ ├─centos-swap swap              3f2904bc-792e-4606-b0af-51bcaee5779c   
│ └─centos-root xfs               0dabeafe-43fe-4f98-9846-eeb2b66dadaa   /
└─vda1          xfs               fecac8e2-cde0-4e2a-86ac-5135aaba25b3   /boot
```

安装git：
```bash
yum install git -y
```

下载rook ceph Git项目仓库（用于安装）：
```bash
git clone https://github.com/rook/rook.git
```

切换标签带V1.13.1版本：
```bash
git checkout v1.13.1
```
### 4. 安装Operator
安装Operator基本不需要过多定制化修改，以下的安装方式可以直接用于生产环境。
```bash
cd rook/deploy/examples/

kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```
完成后检查Pod的运行状态
```bash
kubectl -n rook-ceph get pod
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-7c9659dcab-fh3ui   1/1     Running   0          42s
```
### 5. 创建ceph集群
```bash
kubectl create -f cluster.yaml

# cluster.yaml文件中有很多定制化的参数，上述部署方式使用的是默认配置。
# 比如可以用于设置mon，mgr节点的数量
# 可以用于是否安装图形化界面，设置监控组件，设置污点和容忍等
```
检查Pod运行状态，osd pod的数量取决于集群中的节点数量和配置的设备数量。对于上述默认的cluster.yaml，将为每个节点上找到的每个可用设备创建一个OSD。
```bash
kubectl get po -n rook-ceph -o wide
NAME                                                              READY   STATUS      RESTARTS        AGE     IP              NODE                   NOMINATED NODE   READINESS GATES
csi-cephfsplugin-bf5vp                                            2/2     Running     1 (4m ago)      4m34s   9.30.247.102    yiqi-rancher-worker5   <none>           <none>
csi-cephfsplugin-j9k5q                                            2/2     Running     1 (4m ago)      4m34s   9.30.211.215    yiqi-rancher-worker1   <none>           <none>
csi-cephfsplugin-k6c75                                            2/2     Running     1 (4m ago)      4m34s   9.30.161.76     yiqi-rancher-worker3   <none>           <none>
csi-cephfsplugin-nxmdh                                            2/2     Running     0               4m34s   9.30.246.169    yiqi-rancher-worker4   <none>           <none>
csi-cephfsplugin-provisioner-56956bc974-8vvfk                     5/5     Running     0               4m34s   10.42.53.134    yiqi-rancher-worker4   <none>           <none>
csi-cephfsplugin-provisioner-56956bc974-l8f5p                     5/5     Running     1 (3m59s ago)   4m34s   10.42.240.6     yiqi-rancher-worker5   <none>           <none>
csi-cephfsplugin-tv5kw                                            2/2     Running     1 (4m ago)      4m34s   9.30.160.224    yiqi-rancher-worker2   <none>           <none>
csi-rbdplugin-fnqst                                               2/2     Running     1 (4m ago)      4m34s   9.30.160.224    yiqi-rancher-worker2   <none>           <none>
csi-rbdplugin-mvvrf                                               2/2     Running     0               4m34s   9.30.246.169    yiqi-rancher-worker4   <none>           <none>
csi-rbdplugin-pjltf                                               2/2     Running     1 (4m ago)      4m34s   9.30.247.102    yiqi-rancher-worker5   <none>           <none>
csi-rbdplugin-provisioner-cc97cf4b9-k5frj                         5/5     Running     1 (4m ago)      4m34s   10.42.65.6      yiqi-rancher-worker3   <none>           <none>
csi-rbdplugin-provisioner-cc97cf4b9-zq2r4                         5/5     Running     2 (3m56s ago)   4m34s   10.42.208.133   yiqi-rancher-worker1   <none>           <none>
csi-rbdplugin-qf4ll                                               2/2     Running     1 (4m ago)      4m34s   9.30.211.215    yiqi-rancher-worker1   <none>           <none>
csi-rbdplugin-xr82v                                               2/2     Running     1 (4m ago)      4m34s   9.30.161.76     yiqi-rancher-worker3   <none>           <none>
rook-ceph-crashcollector-yiqi-rancher-worker1.fyre.ibm.comwsxkm   1/1     Running     0               3m4s    10.42.208.138   yiqi-rancher-worker1   <none>           <none>
rook-ceph-crashcollector-yiqi-rancher-worker2.fyre.ibm.comwvzxh   1/1     Running     0               3m25s   10.42.128.21    yiqi-rancher-worker2   <none>           <none>
rook-ceph-crashcollector-yiqi-rancher-worker3.fyre.ibm.comc5cdq   1/1     Running     0               3m50s   10.42.65.8      yiqi-rancher-worker3   <none>           <none>
rook-ceph-crashcollector-yiqi-rancher-worker4.fyre.ibm.comjrv75   1/1     Running     0               3m21s   10.42.53.139    yiqi-rancher-worker4   <none>           <none>
rook-ceph-crashcollector-yiqi-rancher-worker5.fyre.ibm.comqbf4n   1/1     Running     0               3m22s   10.42.240.13    yiqi-rancher-worker5   <none>           <none>
rook-ceph-exporter-yiqi-rancher-worker1.fyre.ibm.com-54bddqm5mb   1/1     Running     0               3m4s    10.42.208.136   yiqi-rancher-worker1   <none>           <none>
rook-ceph-exporter-yiqi-rancher-worker2.fyre.ibm.com-767bc8lkfq   1/1     Running     0               3m21s   10.42.128.22    yiqi-rancher-worker2   <none>           <none>
rook-ceph-exporter-yiqi-rancher-worker3.fyre.ibm.com-6fdfdkwlth   1/1     Running     0               3m50s   10.42.65.9      yiqi-rancher-worker3   <none>           <none>
rook-ceph-exporter-yiqi-rancher-worker4.fyre.ibm.com-5ddb5zk9xg   1/1     Running     0               3m21s   10.42.53.140    yiqi-rancher-worker4   <none>           <none>
rook-ceph-exporter-yiqi-rancher-worker5.fyre.ibm.com-5d4b7gn2ch   1/1     Running     0               3m18s   10.42.240.15    yiqi-rancher-worker5   <none>           <none>
rook-ceph-mgr-a-85945fd8d4-m2sbg                                  3/3     Running     0               4m6s    10.42.128.14    yiqi-rancher-worker2   <none>           <none>
rook-ceph-mgr-b-7f95644479-6xqhr                                  3/3     Running     0               4m6s    10.42.240.7     yiqi-rancher-worker5   <none>           <none>
rook-ceph-mon-a-596c6cbfc5-8nxdj                                  2/2     Running     0               4m57s   10.42.240.5     yiqi-rancher-worker5   <none>           <none>
rook-ceph-mon-b-6f9c6f46c9-g69w8                                  2/2     Running     0               4m33s   10.42.128.13    yiqi-rancher-worker2   <none>           <none>
rook-ceph-mon-c-5c889df47f-488cm                                  2/2     Running     0               4m20s   10.42.65.7      yiqi-rancher-worker3   <none>           <none>
rook-ceph-operator-6c9669dcbd-k5qmz                               1/1     Running     0               25m     10.42.65.4      yiqi-rancher-worker3   <none>           <none>
rook-ceph-osd-0-6846c64444-jv7k4                                  2/2     Running     0               3m25s   10.42.128.18    yiqi-rancher-worker2   <none>           <none>
rook-ceph-osd-1-7dbcd4bdc-w42tw                                   2/2     Running     0               3m24s   10.42.65.11     yiqi-rancher-worker3   <none>           <none>
rook-ceph-osd-10-69cd6c5b7-l7s7b                                  2/2     Running     0               3m22s   10.42.240.14    yiqi-rancher-worker5   <none>           <none>
rook-ceph-osd-11-646d597b-m6ptn                                   2/2     Running     0               3m21s   10.42.53.138    yiqi-rancher-worker4   <none>           <none>
rook-ceph-osd-12-7f4458595b-cdpsr                                 2/2     Running     0               3m4s    10.42.208.139   yiqi-rancher-worker1   <none>           <none>
rook-ceph-osd-13-88b4c74f7-xl6nj                                  2/2     Running     0               3m4s    10.42.208.135   yiqi-rancher-worker1   <none>           <none>
rook-ceph-osd-14-89c7fddd-cgdvw                                   2/2     Running     0               3m4s    10.42.208.137   yiqi-rancher-worker1   <none>           <none>
rook-ceph-osd-2-659fbf966f-2m8zc                                  2/2     Running     0               3m22s   10.42.240.11    yiqi-rancher-worker5   <none>           <none>
rook-ceph-osd-3-5d5cbcd9f5-w8qv9                                  2/2     Running     0               3m22s   10.42.53.137    yiqi-rancher-worker4   <none>           <none>
rook-ceph-osd-4-9f5cb68fb-pp2g6                                   2/2     Running     0               3m25s   10.42.128.19    yiqi-rancher-worker2   <none>           <none>
rook-ceph-osd-5-59d79d7679-j2h9r                                  2/2     Running     0               3m24s   10.42.65.13     yiqi-rancher-worker3   <none>           <none>
rook-ceph-osd-6-5799c4b877-gzgdn                                  2/2     Running     0               3m22s   10.42.240.12    yiqi-rancher-worker5   <none>           <none>
rook-ceph-osd-7-6897c7445c-kctmf                                  2/2     Running     0               3m22s   10.42.53.136    yiqi-rancher-worker4   <none>           <none>
rook-ceph-osd-8-796c48c59f-2j5fl                                  2/2     Running     0               3m25s   10.42.128.20    yiqi-rancher-worker2   <none>           <none>
rook-ceph-osd-9-fff5df8c7-sf7j7                                   2/2     Running     0               3m24s   10.42.65.12     yiqi-rancher-worker3   <none>           <none>
rook-ceph-osd-prepare-22ca3ed4ba60ff130925b1da1d599303-ndtrc      0/1     Completed   0               2m30s   10.42.65.14     yiqi-rancher-worker3   <none>           <none>
rook-ceph-osd-prepare-49eb3309ba3821b41e4320db43c5f1f1-ktxh7      0/1     Completed   0               2m37s   10.42.208.140   yiqi-rancher-worker1   <none>           <none>
rook-ceph-osd-prepare-4d2d5eab315cc094c1f72a4485297888-xg24j      0/1     Completed   0               2m27s   10.42.53.141    yiqi-rancher-worker4   <none>           <none>
rook-ceph-osd-prepare-8545b28a9ad22acf59fd2168d50c1a9c-mlj2l      0/1     Completed   0               2m24s   10.42.240.16    yiqi-rancher-worker5   <none>           <none>
rook-ceph-osd-prepare-b4a0cdf88b658e7c811ab8a320cbd626-f59kr      0/1     Completed   0               2m33s   10.42.128.24    yiqi-rancher-worker2   <none>           <none>
```
创建过程如果遇到问题可以查看rook-ceph-operator的Pod的日志。
### 6. 验证ceph集群
为了验证集群处于健康状态， 需要使用[Rook工具箱](https://rook.io/docs/rook/v1.13/Troubleshooting/ceph-toolbox/)

```bash
kubectl create -f toolbox.yaml

检查Pod状态
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools

通过命令行进入Pod
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

#可以通过如下ceph命令查看集群的状态，例如
ceph status
ceph osd status
ceph df
rados df

# ceph status显示的当前集群状态
bash-4.4$ ceph status
  cluster:
	id: 	b6c450b1-4406-44f9-8ebd-e0277c8d8f7e
	health: HEALTH_WARN
        	1 daemons have recently crashed
 
  services:
	mon: 3 daemons, quorum a,b,c (age 20m)
	mgr: a(active, since 18m), standbys: b
	osd: 15 osds: 15 up (since 18m), 15 in (since 19m)
 
  data:
	pools:   1 pools, 1 pgs
	objects: 2 objects, 449 KiB
	usage:   399 MiB used, 2.9 TiB / 2.9 TiB avail
	pgs: 	1 active+clean

```
### 7. 使用存储
Ceph提供三种类型的存储接口: 块存储（Block）、共享文件系统（Shared Filesystem）、对象存储（Object）。

下面演示对于使用Rook部署和管理的Ceph集群，如何使用这三种存储。

通过Rook使用Ceph提供的三种存储类型以及它们的用途如下:
 - 块存储（Block）适用于为单个 Pod 提供读写一致性（RWO）的存储
 - CephFS 共享文件系统（Shared Filesystem）适用于多个Pod之间共享读写（RWX）的存储
 - 对象存储（Object）提供了一个可通过内部或外部的Kubernetes集群的S3端点访问的存储

#### 7.1 使用块存储(Block)
块存储允许单个Pod挂载存储。本指南介绍了如何使用Rook启用的持久卷，在Kubernetes上创建一个简单的多层Web应用程序。

1.创建Storage Class
在Rook可以提供存储之前，需要创建StorageClass和CephBlockPool CR。这将使Kubernetes在提供持久卷时与Rook进行交互操作。

> 这个示例需要每个节点至少有1个 OSD，并且每个 OSD 需要位于3个不同的节点上。每个OSD必须位于不同的节点上，因为 failureDomain 被设置为 host，并且 replicated.size 被设置为 3。这里多说一句，本次案例中使用的Ceph数据高可用方式为多副本。Ceph中还有一种方式是纠错编码Erasure Coding。可以使用“deploy/examples/csi/rbd/storageclass-ec.yaml”这个文件进行部署。

```bash
# 在当前路径 /root/rook/deploy/examples
kubectl apply -f ./csi/rbd/storageclass.yaml 
cephblockpool.ceph.rook.io/replicapool created
storageclass.storage.k8s.io/rook-ceph-block created
```
这个storageclass.yaml文件中包含了StorageClass rook-ceph-block和CephBlockPool replicapool的定义。

如果你在一个名为"rook-ceph"以外的命名空间中部署了Rook Operator，请将该文件中的provisioner中的前缀更改为与你使用的命名空间匹配。例如，如果Rook Operator在命名空间"my-namespace"中运行，则provisioner的值应为my-namespace.rbd.csi.ceph.com。

> 根据Kubernetes的规定，在使用"Retain"回收策略时，任何由PersistentVolume支持的Ceph RBD 镜像将在PersistentVolume被删除后继续存在。这些Ceph RBD镜像需要使用rbd rm命令手动清理。

上面在创建名称为replicapool的CephBlockPool资源时，会自动在Ceph集群中创建名称为replicapool的存储池。这个操作是由Ceph Operator完成的，如果查看发现没有创建这个存储池，可以通过查看Ceph Operator的日志进行问题定位。

以下命令在ToolBox容器中运行的输出说明已经创建了存储池replicapool。
```bash
bash-4.4$ ceph osd lspools
1 .mgr
2 replicapool
```

通过以下命令可以检查Storage Class的状态（nfs-client是我这个集群之前安装的）：
```bash
kubectl get sc
NAME                   PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   6d23h
rook-ceph-block        rook-ceph.rbd.csi.ceph.com                      Delete          Immediate           true                   7m20s
```

我们创建了一个示例应用程序，使用经由Rook提供的块存储，其中包括经由Rook提供的的WordPress和MySQL应用程序。这些应用程序都将使用Rook提供的块存储卷(block volumes)。
```bash
# 在当前路径 /root/rook/deploy/examples
kubectl create -f mysql.yaml
kubectl create -f wordpress.yaml
```

查看PVC
```bash
kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mysql-pv-claim   Bound    pvc-3db54f6f-b4b2-4c72-8134-81841ce99178   20Gi       RWO            rook-ceph-block   3m20s
wp-pv-claim      Bound    pvc-5134d1a7-23c5-4bfe-be02-6d204121a127   20Gi       RWO            rook-ceph-block   3m19s

kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS      REASON   AGE
pvc-3db54f6f-b4b2-4c72-8134-81841ce99178   20Gi       RWO            Delete           Bound    default/mysql-pv-claim   rook-ceph-block            3m19s
pvc-5134d1a7-23c5-4bfe-be02-6d204121a127   20Gi       RWO            Delete           Bound    default/wp-pv-claim      rook-ceph-block            3m19s
```

回到ToolBox容器中检查rbd储存池使用状态
```bash
bash-4.4$ rbd ls -p replicapool
csi-vol-ac06bcbf-5c33-42ea-b41f-e46a4a4d9c77
csi-vol-be9cd19a-4fff-42b3-85f3-9ea021110da0
```
#### 7.2 使用文件存储(CephFS)
CephFS文件系统存储（也称为共享文件系统）可以从多个Pod中以读/写权限挂载。这对于可以使用共享文件系统进行集群化的应用程序可能非常有用。

> CephFS CSI驱动程序使用配额来强制执行所请求的PVC大小。只有较新的Linux内核支持CephFS配额，至少是4.17版本的内核。

Ceph从自Pacific版本(Ceph 16)开始，支持多个文件系统。

1.创建CephFilesystem
通过在CephFilesystem CRD中指定元数据池(metadata pool)、数据池(data pools)和元数据服务(metadata server)的所需设置来创建文件系统。在这里，我们创建的是3个副本的元数据池和3个副本的单个数据池。有关更多选项，请参阅[创建共享文件系统的文档](https://rook.github.io/docs/rook/latest-release/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/)

```bash
# 在当前路径 /root/rook/deploy/examples
kubectl create -f filesystem.yaml
```

查看Pod状态
```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-mds
NAME                                	READY   STATUS	RESTARTS   AGE
rook-ceph-mds-myfs-a-795d4c688f-nj92g   2/2 	Running   0      	104s
rook-ceph-mds-myfs-b-558bb747c7-kpwqd   2/2 	Running   0      	103s
```

要查看文件系统的详细状态，进入Rook toolbox中，使用ceph status查看， 确认输出中包含MDS服务的状态。在这个示例中，有一个处于活动状态的MDS实例，并且还有一个处于备用MDS实例，以防发生故障切换。

```bash
bash-4.4$ ceph -s
  cluster:
	id: 	b6c450b1-4406-44f9-8ebd-e0277c8d8f7e
	health: HEALTH_WARN
        	1 daemons have recently crashed
 
  services:
	mon: 3 daemons, quorum a,b,c (age 86m)
	mgr: a(active, since 84m), standbys: b
	mds: 1/1 daemons up, 1 hot standby  #可以看到相比之前的状态mds服务已经运行起来了
	osd: 15 osds: 15 up (since 84m), 15 in (since 85m)
 
  data:
	volumes: 1/1 healthy
	pools:   4 pools, 81 pgs
	objects: 29 objects, 466 KiB
	usage:   886 MiB used, 2.9 TiB / 2.9 TiB avail
	pgs: 	81 active+clean
 
  io:
	client:   853 B/s rd, 1 op/s rd, 0 op/s wr
 
# 使用ceph osd lspools查看，确认创建了myfs-metadata和myfs-replicated的存储池。 
bash-4.4$ ceph osd lspools
1 .mgr
2 replicapool
3 myfs-metadata 
4 myfs-replicated #
```

2.创建StorageClass
```bash
# 在当前路径 /root/rook/deploy/examples
kubectl create -f ./csi/cephfs/storageclass.yaml

#检查StorageClass的状态，可以看到rook-cephfs已经建立起来了。
kubectl get sc
NAME                   PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   7d
rook-ceph-block        rook-ceph.rbd.csi.ceph.com                      Delete          Immediate           true                   48m
rook-cephfs            rook-ceph.cephfs.csi.ceph.com                   Delete          Immediate           true                   19s
```

创建一个测试用的Deployment和PVC挂载

```bash
vi testcephps.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: busybox-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  storageClassName: rook-cephfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
spec:
  replicas: 2
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
      - name: busybox-container
        image: busybox:stable-glibc
        command: ["sleep", "3600"]
        volumeMounts:
        - name: data-volume
          mountPath: /data
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: busybox-data-pvc

#创建相关的资源
kubectl apply -f testcephfs.yaml

#检查资源的创建情况
kubectl get pvc
NAME           	STATUS   VOLUME                                 	CAPACITY   ACCESS MODES   STORAGECLASS   AGE
busybox-data-pvc   Bound	pvc-647ecb9f-1a18-484a-9536-ac26cb0a8a03   2Gi    	RWX        	rook-cephfs	9s

kubectl get po
NAME                                           	READY   STATUS	RESTARTS   AGE
busybox-86c4ffc4b6-q6lzn                       	1/1 	Running   0      	41s
busybox-86c4ffc4b6-ttxgz                       	1/1 	Running   0      	41s
```
#### 7.3 使用对象存储(Object)
对象存储为应用程序提供了一个使用S3 API存储和获取数据的接口。

1.配置一个Object Store
Rook具有在Kubernetes中部署对象存储或连接到外部RGW服务的能力。通常情况下，对象存储将由Rook在本地进行配置。或者，如果你有一个现有的带有Rados Gateways的Ceph集群，请参阅[external section文档](https://rook.github.io/docs/rook/latest-release/Storage-Configuration/Object-Storage-RGW/object-storage/#create-a-local-object-store)以从Rook中使用它。

以下示例将创建一个CephObjectStore，该对象存储在集群中启动RGW服务，并提供S3 API。

> 注意: 这个示例至少需要3个bluestore OSD，每个OSD位于不同的节点上。OSDs必须位于不同的节点上，因为failureDomain设置为host，并且erasureCoded块设置要求至少有3个不同的OSD（2个dataChunks + 1个codingChunks）。

```bash
# 在当前路径 /root/rook/deploy/examples
kubectl apply -f object.yaml
cephobjectstore.ceph.rook.io/my-store created
```
创建CephObjectStore后，Rook Ceph Operator将创建所有必要的池和其他资源以启动服务。这可能需要几分钟来完成。

要确认对象存储已配置完成，请等待RGW pod启动：

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-rgw
NAME                                        READY   STATUS    RESTARTS   AGE
rook-ceph-rgw-my-store-a-5c9fcc5674-j85lv   2/2     Running   0          109s
```

通过Toolbox程序检查Ceph集群的状态：
```bash
bash-4.4$ ceph osd lspools
1 .mgr
2 replicapool
3 myfs-metadata
4 myfs-replicated
5 my-store.rgw.buckets.non-ec
6 my-store.rgw.buckets.index
7 my-store.rgw.log
8 my-store.rgw.control
9 my-store.rgw.otp
10 .rgw.root
11 my-store.rgw.meta
12 my-store.rgw.buckets.data
```
Rook Ceph Operator会在Ceph集群中创建名称为my-store.rgw.*的多个存储池。

再查看一下ceph集群的状态:

```bash
bash-4.4$ ceph -s
  cluster:
    id:     b6c450b1-4406-44f9-8ebd-e0277c8d8f7e
    health: HEALTH_WARN
            1 daemons have recently crashed
 
  services:
    mon: 3 daemons, quorum a,b,c (age 5h)
    mgr: a(active, since 5h), standbys: b
    mds: 1/1 daemons up, 1 hot standby
    osd: 15 osds: 15 up (since 5h), 15 in (since 5h)
    rgw: 1 daemon active (1 hosts, 1 zones)
 
  data:
    volumes: 1/1 healthy
    pools:   12 pools, 169 pgs
    objects: 245 objects, 753 KiB
    usage:   1.2 GiB used, 2.9 TiB / 2.9 TiB avail
    pgs:     169 active+clean
 
  io:
    client:   1.3 KiB/s rd, 170 B/s wr, 2 op/s rd, 0 op/s wr
```

2.创建StorageClass

```bash
# 在当前路径 /root/rook/deploy/examples
kubectl create -f storageclass-bucket-delete.yaml

storageclass.storage.k8s.io/rook-ceph-delete-bucket created
```
检查已创建的StorageClass

```bash
kubectl get sc
NAME                      PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)      cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   7d4h
rook-ceph-block           rook-ceph.rbd.csi.ceph.com                      Delete          Immediate           true                   5h26m
rook-ceph-delete-bucket   rook-ceph.ceph.rook.io/bucket                   Delete          Immediate           false                  45s
rook-cephfs               rook-ceph.cephfs.csi.ceph.com                   Delete          Immediate           true                   4h38m
```

根据这个存储类，现在可以通过创建对象存储桶声明Object Bucket Claim（OBC）来请求一个存储桶。当创建OBC时，Rook bucket provisioner将创建一个新的存储桶。请注意，OBC引用了上面创建的存储类。将以下内容保存为object-bucket-claim-delete.yaml（示例命名为 delete，因为使用了Delete的回收策略）：

```bash
# 在当前路径 /root/rook/deploy/examples
kubectl create -f object-bucket-claim-delete.yaml
```
OBC创建成功后，Rook Ceph Operator将创建存储桶，并生成其他文件以实现对存储桶的访问。将以与OBC相同的名称和在相同的命名空间中创建一个密钥和ConfigMap。密钥包含应用程序Pod用于访问存储桶的凭据。ConfigMap包含存储桶端点信息，也会被Pod使用。有关CephObjectBucketClaims的更多详细信息，请参阅[对象存储桶声明文档](https://rook.github.io/docs/rook/latest-release/Storage-Configuration/Object-Storage-RGW/ceph-object-bucket-claim/)

```bash
#查看相关的资源Configmap
kubectl get cm
NAME                 DATA   AGE
ceph-delete-bucket   5      10m

#查看Configmap的具体信息
kubectl get cm ceph-delete-bucket -o yaml
apiVersion: v1
data:
  BUCKET_HOST: rook-ceph-rgw-my-store.rook-ceph.svc
  BUCKET_NAME: ceph-bkt-2e22eaf9-a577-45b2-94ed-e96c04670637
  BUCKET_PORT: "80"
  BUCKET_REGION: ""
  BUCKET_SUBREGION: ""
kind: ConfigMap
metadata:
  creationTimestamp: "2023-12-30T09:42:56Z"
  finalizers:
  - objectbucket.io/finalizer
  labels:
	bucket-provisioner: rook-ceph.ceph.rook.io-bucket
  name: ceph-delete-bucket
  namespace: default
  ownerReferences:
  - apiVersion: objectbucket.io/v1alpha1
	blockOwnerDeletion: true
	controller: true
	kind: ObjectBucketClaim
	name: ceph-delete-bucket
	uid: ae670efd-0db9-4019-b0c3-6fa66aafcefb
  resourceVersion: "32176508"
  uid: 820df543-bc37-41ba-b777-425093cf20d8

#查看Secret的具体信息，可以看到存储桶的访问ID和密码
kubectl get secret ceph-delete-bucket -o yaml
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: RjFEWlhFTTQxQThBUENGOFcxNzY=
  AWS_SECRET_ACCESS_KEY: VGxhZTlRcFJESWU2U3N3OW91MXYyWGxVUFFCUEhuallhVWVCTGZreA==
```

3.使用s5cmd工具访问对象存储
因为这里的k8s集群使用的calico网络，可以直接在Node节点上使用svc ip，这里简单起见直接在K8S集群的控制节点上安装s5cmd命令行工具。

```bash
wget https://github.com/peak/s5cmd/releases/download/v2.2.2/s5cmd_2.2.2_Linux-64bit.tar.gz

tar -xvf s5cmd_2.2.2_Linux-64bit.tar.gz
CHANGELOG.md
LICENSE
README.md
s5cmd

sudo mv s5cmd /usr/local/bin/

#最后通过 --help 既可以查看帮助命令
s5cmd --help
NAME:
   s5cmd - Blazing fast S3 and local filesystem execution tool

USAGE:
   s5cmd [global options] command [command options] [arguments...]
```

接下来配置s5cmd工具访问对象存储凭据:

```bash
export AWS_ACCESS_KEY_ID=$(kubectl -n default get secret ceph-delete-bucket -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 --decode)
export AWS_SECRET_ACCESS_KEY=$(kubectl -n default get secret ceph-delete-bucket -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 --decode)

mkdir ~/.aws
cat > ~/.aws/credentials << EOF
[default]
aws_access_key_id = ${AWS_ACCESS_KEY_ID}
aws_secret_access_key = ${AWS_SECRET_ACCESS_KEY}
EOF
```
查看rook-ceph-rgw-my-store这个Service的IP:

```bash
k get svc rook-ceph-rgw-my-store  -n rook-ceph
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
rook-ceph-rgw-my-store   ClusterIP   10.43.65.160   <none>        80/TCP    103m
```
使用s5cmd访问存储桶，列出当前凭据可以访问的所有桶:

```bash
s5cmd --endpoint-url  http://10.43.65.160 ls
2023/12/30 09:42:56  s3://ceph-bkt-2e22eaf9-a577-45b2-94ed-e96c04670637
```

> 注这里使用Service IP 访问rook-ceph-rgw-my-store只是为了测试。生产环境中如果从K8S集群内访问的话，使用Service的DNS名称，如果从集群外部访问推荐使用Ingress将其暴露到集群外部。
### 8. 访问Ceph Dashboard
通过使用Ceph Dashboard可以查看集群的状态。使用Rook部署的Ceph集群已经默认启用了Ceph Dashboard。

创建一个拥有NodePort对外暴露方式的Service

```bash
kubectl apply -f dashboard-external-https.yaml
service/rook-ceph-mgr-dashboard-external-https created

#可以看到当前NodePort对外的端口为57482
kg svc -n rook-ceph
NAME                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
rook-ceph-exporter                       ClusterIP   10.43.179.89    <none>        9926/TCP            7h38m
rook-ceph-mgr                            ClusterIP   10.43.11.237    <none>        9283/TCP            7h38m
rook-ceph-mgr-dashboard                  ClusterIP   10.43.234.132   <none>        8443/TCP            7h38m
rook-ceph-mgr-dashboard-external-https   NodePort    10.43.207.88    <none>        8443:57482/TCP      11m
rook-ceph-mon-a                          ClusterIP   10.43.205.40    <none>        6789/TCP,3300/TCP   7h39m
rook-ceph-mon-b                          ClusterIP   10.43.180.30    <none>        6789/TCP,3300/TCP   7h39m
rook-ceph-mon-c                          ClusterIP   10.43.166.149   <none>        6789/TCP,3300/TCP   7h38m
rook-ceph-rgw-my-store                   ClusterIP   10.43.65.160    <none>        80/TCP              148m
```

Ceph Dashboard admin用户的命名可以通过下面的命令查看:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```
登录图形化界面
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/805f11403ca6472db43d6050e00822f8.png)
