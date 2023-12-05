## Kubernetes集群安装高可用postgresql
Bitnami 提供的 `postgresql-ha` 解决方案是一个预配置的、高可用的 PostgreSQL 集群配置，通常部署在 Kubernetes 环境中。它使用了一些关键技术和组件来实现数据库的高可用性。，Bitnami `postgresql-ha` 主要采用以下构建方式：

1. **PostgreSQL 集群**：这是核心部分，通常包含一个主（Primary）数据库和一个或多个从（Standby）数据库。这种设置支持主从复制，其中从数据库实时复制主数据库的数据。

2. **自动故障转移**：在主数据库发生故障时，系统会自动将其中一个从数据库提升为新的主数据库，以确保服务的持续可用性。

3. **Pgpool-II**：Bitnami 的 `postgresql-ha` 使用 Pgpool-II 作为数据库连接池和负载均衡器。Pgpool-II 处理客户端连接，提供负载均衡和连接池功能，同时也支持自动故障转移和读写分离。

4. **持久化存储**：为了保证数据的持久性和稳定性，Bitnami 的解决方案通常使用持久化存储，如 Kubernetes 的持久卷（Persistent Volumes，PVs）和持久卷声明（Persistent Volume Claims，PVCs）。

5. **监控和日志记录**：集成的监控和日志记录机制，以确保集群的健康状况可以被实时监控，并在出现问题时可以迅速响应。

6. **配置和管理**：Bitnami 的 Helm chart 提供了灵活的配置选项，允许用户根据具体需求调整数据库设置、资源分配、复制策略等。

7. **安全性**：通常包括网络策略、访问控制和加密选项来保护数据和通信。

8. **备份和恢复**：可能包括对数据库备份和恢复的支持，以确保数据的安全性。

使用这样的架构，Bitnami 的 `postgresql-ha` 解决方案能够为企业级应用提供可靠的、高可用的数据库服务，同时充分利用了 Kubernetes 平台的特性，如易于扩展、自我修复和声明式配置。

如果想进一步了解pgpool实现高可用postgresql数据库的架构原理，请参考：[pgpool-II高可用配置讲解](https://www.bilibili.com/video/BV1rY4y1r7vA/?spm_id_from=333.337.search-card.all.click&vd_source=c29814fb5a844335f87ca53d3cfb0ac5) 
### 1.通过Helm Chart安装高可用Postgresql集群
#### 1.1 前提条件
a. Kubernetes版本>=1.23，Kubernetes的安装请参考：[Kubeadm安装K8s1.26集群](https://blog.csdn.net/weixin_46660849/article/details/130580689)

b. Helm版本>=v3.8.0，其中 Helm 的安装请参考：[Helm Install](https://helm.sh/docs/intro/install/)

c.需要有默认的StorageClass，具体准备流程参考：[Kubernetes安装StorageClass](https://blog.csdn.net/weixin_46660849/article/details/130881771)
#### 1.2 安装流程
a.创建安装目录
```bash
#切换到当前用户根目录并建立logging文件夹
cd ~ && mkdir postgresql-ha

cd postgresql-ha
```
b.创建logging名字空间，独立的名字空间有助于资源管理
```bash
kubectl create ns pg
```
c.添加elastic官方repo仓库
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
d.定制化
如果需要做某些定制化需求请参考官方 values.yaml 文件，其中每个参数都会有详细描述: [官网参考](https://artifacthub.io/packages/helm/bitnami/postgresql-ha) 
```bash
helm pull bitnami/postgresql-ha --version 12.3.2

tar -xvf postgresql-ha-12.3.2.tgz

cd postgresql-ha

#（非必要）以下的操作主要是修改postgresql：默认的pgpool容器副本数，持久化存储卷大小，NodePort对外暴露方式。当然如果需要还可以修改数据库名，数据库密码，登录用户名等信息，也可以通过ingress实现对外7层代理，当然如果需要还可以修改镜像地址，毕竟从外网下载镜像稳定性较差。
vi values.yaml
# 1.全局搜索“pgpool.replicaCount”

## @param pgpool.replicaCount The number of replicas to deploy
##
replicaCount: 2 #修改pgpool的数量增强高可用性

# 2.全局搜索“persistence.size”

## @param persistence.size Persistent Volume Claim size
##
size: 8Gi #默认8G比较小，在真实生产环境可以根据实际需求修改。

# 3.全局搜索“service.type”，找到如下内容：
  type: NodePort #修改为NodePort方式
  ## @param service.ports.postgresql PostgreSQL port
  ##
  ports:
    postgresql: 5432
  ## @param service.portName PostgreSQL service port name
  ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#multi-port-services
  ##
  portName: postgresql
  ## @param service.nodePorts.postgresql Kubernetes service nodePort
  ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
  ##
  nodePorts:
    postgresql: "15432" #选择一个合适的NodePort端口
```
e.安装
```bash
#执行安装命令
cd ~/postgresql-ha/

helm install pg-ha ./postgresql-ha -n pg

#成功后提示
NAME: pg-ha
LAST DEPLOYED: Tue Dec  5 04:36:18 2023
NAMESPACE: pg
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql-ha
CHART VERSION: 12.3.2
APP VERSION: 16.1.0
** Please be patient while the chart is being deployed **
PostgreSQL can be accessed through Pgpool via port 5432 on the following DNS name from within your cluster:

    pg-ha-postgresql-ha-pgpool.pg.svc.cluster.local

Pgpool acts as a load balancer for PostgreSQL and forward read/write connections to the primary node while read-only connections are forwarded to standby nodes.

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace pg pg-ha-postgresql-ha-postgresql -o jsonpath="{.data.password}" | base64 -d)

To get the password for "repmgr" run:

    export REPMGR_PASSWORD=$(kubectl get secret --namespace pg pg-ha-postgresql-ha-postgresql -o jsonpath="{.data.repmgr-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run pg-ha-postgresql-ha-client --rm --tty -i --restart='Never' --namespace pg --image docker.io/bitnami/postgresql-repmgr:16.1.0-debian-11-r11 --env="PGPASSWORD=$POSTGRES_PASSWORD"  \
        --command -- psql -h pg-ha-postgresql-ha-pgpool -p 5432 -U postgres -d postgres

To connect to your database from outside the cluster execute the following commands:

    export NODE_IP=$(kubectl get nodes --namespace pg -o jsonpath="{.items[0].status.addresses[0].address}")
    export NODE_PORT=$(kubectl get --namespace pg -o jsonpath="{.spec.ports[0].nodePort}" services pg-ha-postgresql-ha-pgpool
    PGPASSWORD="$POSTGRES_PASSWORD" psql -h $NODE_IP -p $NODE_PORT -U postgres -d postgres

#查看运行状态
kubectl get po -n pg
NAME                                          READY   STATUS    RESTARTS   AGE
pg-ha-postgresql-ha-pgpool-58468c7bff-jg9kz   1/1     Running   0          3m41s
pg-ha-postgresql-ha-pgpool-58468c7bff-lhf5p   1/1     Running   0          3m41s
pg-ha-postgresql-ha-postgresql-0              1/1     Running   0          3m41s
pg-ha-postgresql-ha-postgresql-1              1/1     Running   0          3m41s
pg-ha-postgresql-ha-postgresql-2              1/1     Running   0          3m41s
```
f.测试连接
```bash
#获取数据库连接密码
kubectl get secret --namespace pg pg-ha-postgresql-ha-postgresql -o jsonpath="{.data.password}" | base64 -d
f6PEWNNTec  #别用我的，咱们不一样

#检查对外暴露的端口
kubectl get svc -n pg
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
pg-ha-postgresql-ha-pgpool                NodePort    172.16.86.61    <none>        5432:15432/TCP   12m
pg-ha-postgresql-ha-postgresql            ClusterIP   172.16.19.127   <none>        5432/TCP         12m
pg-ha-postgresql-ha-postgresql-headless   ClusterIP   None            <none>        5432/TCP         12m

#使用数据库连接工具，我使用的是pgAdmin
```
可以看到已经能够连接成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/c6566d4ef2a045ac909d6e9d0133c6e0.png)
g.错误处理
我这里做过多次安装，有的机器可能遇到如下错误

> password authentication failed for user "postgres"; User "postgres" has no password assigned.

这个问题是由于postgres数据库启动太慢导致的，适当增加livenessProbe，readinessProbe，startupProbe的initialDelaySeconds数值即可。
```bash
# 1.全局搜索“postgresql.livenessProbe.initialDelaySeconds”

# 2.全局搜索“postgresql.readinessProbe.initialDelaySeconds”

# 3.全局搜索“postgresql.startupProbe.initialDelaySeconds”
```
