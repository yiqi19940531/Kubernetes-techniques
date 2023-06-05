## Helm Chart安装Prometheus&Grafana
如果只想了解安装请查看“kube-prometheus-stack V64.6.0安装”章节。
### Prometheus监控解决方案介绍
Prometheus是一种开源的系统监控和警报工具包，由SoundCloud公司开发。它被设计为在动态环境中进行可靠的服务发现，并支持多种数据模型。Prometheus是云原生计算基金会（CNCF）的一部分，与Kubernetes非常契合。Prometheus提供了全套的服务发现&数据收集&数据分析&数据展示&数据告警&自定义扩展接口，Prometheus是目前Kubernetes开源领域监控方案的行业标准。为在Kubernetes环境中监控和理解系统提供了大量工具。以下是Prometheus在Kubernetes监控领域的作用和功能：

 - **数据模型和查询语言**：Prometheus使用多维数据模型和强大的PromQL查询语言，可以基于时间序列数据进行复杂的查询和数据分析。
 - **服务发现**：Prometheus自动发现新的服务和容器，使其在动态环境中（如Kubernetes）非常有用。对于Kubernetes，Prometheus可以发现新的节点、服务、容器等。
 - **多种数据导出格式**：Prometheus支持多种数据导出格式，允许从不同的系统和服务收集数据。
 - **强大的警报系统**：Prometheus具有强大的警报系统，可以根据定义的规则向用户发送警报。警报可以通过多种方式发送，包括电子邮件、Slack、PagerDuty等。
 - **Grafana集成**：Prometheus可以很好地与Grafana集成，为可视化监控数据提供强大的工具。
 - **高效性能**：Prometheus使用一种高效的内置存储机制，可以在单个服务器上处理大量的时间序列数据。
### Prometheus架构
![在这里插入图片描述](https://img-blog.csdnimg.cn/30b368bef78a410bbd9d14cacaa8ca35.png)
 - Prometheus Server ，监控、告警平台核心，抓取目标端监控数据，生成聚合数据，存储时间序列数据
 - exporter，由被监控的对象提供，提供API暴漏监控对象的指标，供 prometheus 抓取 node-exporter、blackbox-exporter、redis-exporter、custom-exporter 等对象的指标数据
 - pushgateway，提供一个网关地址，外部数据可以推送到该网关，prometheus也会从该网关拉取数据
 - Alertmanager，接收Prometheus发送的告警并对于告警进行一系列的处理后发送给指定的目标
 - Grafana：配置数据源，图标方式展示数据

我重点介绍基于Helm Chart的安装实现，关于更多功能方面的细节建议观看以下视频：
[介绍视频1](https://www.bilibili.com/video/BV1GU4y1e7HK?p=23&vd_source=c29814fb5a844335f87ca53d3cfb0ac5)
[介绍视频2](https://www.bilibili.com/video/BV17v4y1H76R/?spm_id_from=333.337.search-card.all.click&vd_source=c29814fb5a844335f87ca53d3cfb0ac5)
### 1. kube-prometheus-stack V64.6.0安装 
[官网参考](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack#configuration) 
以下介绍的安装实现会从实际生产角度去做一些考量，之所以需要StorageClass是为了保证数据持久化。
#### 1.1 前提条件
a. Kubernetes版本>=1.16

b. Helm版本>=v3.2.0，其中 Helm 的安装请参考：[Helm Install](https://helm.sh/docs/intro/install/)

c.需要有默认的StorageClass，具体准备流程参考：[Kubernetes安装StorageClass](https://blog.csdn.net/weixin_46660849/article/details/130881771)
#### 1.2 安装流程
a.创建安装目录
```bash
#切换到当前用户根目录并建立monitor文件夹
cd ~ && mkdir monitor

cd monitor
```
b.创建monitor名字空间，独立的名字空间有助于资源管理
```bash
kubectl create ns monitor
```
c.添加prometheus官方repo仓库
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
d.下载 kube-prometheus-stack V64.6.0 文件，下载相关的文件有助于我们更好的管理项目，安装前会修改 values.yaml 文件
```bash
helm pull prometheus-community/kube-prometheus-stack --version 46.6.0

tar -xvf kube-prometheus-stack-46.6.0.tgz

cd kube-prometheus-stack
```
e.修改 values.yaml 内配置，kube-prometheus-stack 的 values.yaml 可配置的内容很多有3000多行，强烈建议将 values.yaml 的内容导入到某些图形画的文本编辑器内进行编辑(例如：vscode)，接下来介绍几个常用的修改：
```bash
# 修改service的暴露类型为NodePort
全局搜索“type: ClusterIP”关键字，将其替换为“type: NodePort”
# 与其相关的位置大多会有Ingress的设置，如果需要将服务通过Ingress的方式暴露，可以进行相关的设置。

# 修改存储类型为StorageClass
全局搜索“storageClass”关键字，共有3处

1. Alertmanager 使用的，改为如下配置，注意 storageClassName 需要是集群中 storageClass 的实际名称
    ## Storage is the definition of how storage will be used by the Alertmanager instances.
    ## ref: https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/storage.md
    ##
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-client
          accessModes: ['ReadWriteOnce']
          resources:
            requests:
              storage: 50Gi

2. Prometheus 持久化数据，改为如下配置，注意 storageClassName 需要是集群中 storageClass 的实际名称
    ## Prometheus StorageSpec for persistent data
    ## ref: https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/storage.md
    ##
    storageSpec:
      ## Using PersistentVolumeClaim
      ##
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-client
          accessModes: ['ReadWriteOnce']
          resources:
            requests:
              storage: 50Gi

# 修改镜像地址
全局搜索“image:”关键字
# 有些镜像来自于quay.io，国内下载受限，需要修改为可以下载的地址（私有仓库或配置阿里镜像加速）
```
values.yaml 文件内容过长，不具体列出，感兴趣的可以查看我的 [上传文件](https://download.csdn.net/download/weixin_46660849/87865765)
values.yaml 中的其他参数设置请参考 [官网说明](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack#configuration) 
f. Helm Chart执行安装
```bash
helm install monitor ~/monitor/kube-prometheus-stack -n monitor
```
### 2. 功能验证
#### 2.1 查看相关资源
```bash
kubectl get po,svc,pvc,secret -n monitor
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-monitor-kube-prometheus-st-alertmanager-0   2/2     Running   0          58m
pod/monitor-grafana-5c955b9765-2h7xd                         3/3     Running   0          58m
pod/monitor-kube-prometheus-st-operator-5bcdfff95c-mtdl2     1/1     Running   0          58m
pod/monitor-kube-state-metrics-6fd4d95d46-7tqfh              1/1     Running   0          58m
pod/monitor-prometheus-node-exporter-7hk2q                   1/1     Running   0          58m
pod/monitor-prometheus-node-exporter-h89ll                   1/1     Running   0          58m
pod/monitor-prometheus-node-exporter-jp9cp                   1/1     Running   0          58m
pod/monitor-prometheus-node-exporter-r7pd6                   1/1     Running   0          58m
pod/prometheus-monitor-kube-prometheus-st-prometheus-0       2/2     Running   0          58m

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   58m
service/monitor-grafana                           NodePort    10.233.47.220   <none>        80:31721/TCP                 58m
service/monitor-kube-prometheus-st-alertmanager   NodePort    10.233.245.52   <none>        9093:30903/TCP               58m
service/monitor-kube-prometheus-st-operator       NodePort    10.233.99.28    <none>        443:30443/TCP                58m
service/monitor-kube-prometheus-st-prometheus     NodePort    10.233.61.216   <none>        9090:30090/TCP               58m
service/monitor-kube-state-metrics                ClusterIP   10.233.211.59   <none>        8080/TCP                     58m
service/monitor-prometheus-node-exporter          ClusterIP   10.233.82.51    <none>        9100/TCP                     58m
service/prometheus-operated                       ClusterIP   None            <none>        9090/TCP                     58m

NAME                                                                                                                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/alertmanager-monitor-kube-prometheus-st-alertmanager-db-alertmanager-monitor-kube-prometheus-st-alertmanager-0   Bound    pvc-35ffe0d7-bdf9-4bfb-8a1d-a66f45e68b35   50Gi       RWO            nfs-client     58m
persistentvolumeclaim/prometheus-monitor-kube-prometheus-st-prometheus-db-prometheus-monitor-kube-prometheus-st-prometheus-0           Bound    pvc-9d4a2ab7-44b0-40e4-8248-03e26ff16f49   50Gi       RWO            nfs-client     58m

NAME                                                                       TYPE                                  DATA   AGE
secret/alertmanager-monitor-kube-prometheus-st-alertmanager                Opaque                                1      58m
secret/alertmanager-monitor-kube-prometheus-st-alertmanager-generated      Opaque                                1      58m
secret/alertmanager-monitor-kube-prometheus-st-alertmanager-tls-assets-0   Opaque                                0      58m
secret/alertmanager-monitor-kube-prometheus-st-alertmanager-web-config     Opaque                                1      58m
secret/default-token-vzn75                                                 kubernetes.io/service-account-token   3      112m
secret/monitor-grafana                                                     Opaque                                3      58m
secret/monitor-grafana-token-ttrxz                                         kubernetes.io/service-account-token   3      58m
secret/monitor-kube-prometheus-st-admission                                Opaque                                3      75m
secret/monitor-kube-prometheus-st-alertmanager-token-ff894                 kubernetes.io/service-account-token   3      58m
secret/monitor-kube-prometheus-st-operator-token-l5fxx                     kubernetes.io/service-account-token   3      58m
secret/monitor-kube-prometheus-st-prometheus-token-twvtr                   kubernetes.io/service-account-token   3      58m
secret/monitor-kube-state-metrics-token-rckng                              kubernetes.io/service-account-token   3      58m
secret/monitor-prometheus-node-exporter-token-t4xjq                        kubernetes.io/service-account-token   3      58m
secret/prometheus-monitor-kube-prometheus-st-prometheus                    Opaque                                1      58m
secret/prometheus-monitor-kube-prometheus-st-prometheus-tls-assets-0       Opaque                                1      58m
secret/prometheus-monitor-kube-prometheus-st-prometheus-web-config         Opaque                                1      58m
secret/sh.helm.release.v1.monitor.v1                                       helm.sh/release.v1                    1      59m
```
#### 2.2 修改service monitor-grafana 的暴露方式
默认monitor-grafana的暴露方式为ClusterIP（其内容并不受 values.yaml 文件控制）
```bash
kubectl edit svc monitor-grafana  -n monitor

# 修改其中spec.type内容为 NodePort
```
#### 2.3 获取 grafana 界面用户名密码
用户名密码存放在secret/monitor-grafana中
```bash
# 获取用户名
kubectl -n monitor get secret monitor-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# 获取密码
kubectl -n monitor get secret monitor-grafana -o jsonpath="{.data.admin-user}" | base64 --decode ; echo
```
#### 2.4 登录Prometheus界面
对应service monitor-kube-prometheus-st-prometheus 的NodePort端口。进入Target页面：
![在这里插入图片描述](https://img-blog.csdnimg.cn/58c8a3e805774f1c8f4059a149b0442e.png)
可以看到 api-server、coredns、kubelete、node-expoter 等关键组件都已被添加。
#### 2.5 登录Grafana界面
需要用到2.2步中修改的NodePort端口，2.3步中获取的用户名、密码

a.Plugins 界面已添加常用的项目
![在这里插入图片描述](https://img-blog.csdnimg.cn/f40a9470cce54afe868c55da19ade26c.png)
b.Dashboards 界面已添加常用的项目
![在这里插入图片描述](https://img-blog.csdnimg.cn/fa3788d4b089441483c654a5fb72b5c1.png)
c.查看集群内pod的资源状态
![在这里插入图片描述](https://img-blog.csdnimg.cn/5ee1cc705d5f4761b9d32318d040097d.png)
