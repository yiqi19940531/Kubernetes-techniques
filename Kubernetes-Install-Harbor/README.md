## Kubernetes通过Helm Chart安装Harbor仓库并访问验证
### Harbor基本介绍
Harbor是一个开源的企业级Docker和OCI（Open Container Initiative）镜像仓库，用于存储、分发和管理容器镜像。它提供了一个安全可靠的方式来管理和共享容器镜像，适用于构建和部署容器化应用程序的环境。
Harbor镜像仓库是当前企业级镜像仓库的首选。

Harbor镜像仓库的主要特点和功能：

 - 镜像存储和管理：Harbor允许用户将Docker和OCI镜像上传到仓库中进行存储和管理。它提供了版本控制、标签管理和元数据存储，可以方便地浏览、搜索和筛选镜像。
 - 安全和权限控制：Harbor具有强大的安全性和权限控制功能。它支持用户认证和授权，可以通过角色和权限来限制用户对镜像的访问和操作。此外，Harbor还提供了漏洞扫描和静态代码分析等安全功能，帮助用户及时发现和修复容器镜像中的安全漏洞。
 - 注册表复制和同步：Harbor支持注册表复制和同步功能，可以将镜像从一个Harbor实例复制到另一个实例，或者与其他Docker Registry进行同步。这使得用户可以在多个环境之间共享和部署容器镜像，提高了镜像的可用性和可靠性。
 - 企业级特性：作为一个企业级镜像仓库，Harbor提供了许多特性来满足企业需求。它支持LDAP/AD集成，可以与现有的用户认证系统进行集成。此外，Harbor还提供了审计日志、报告和统计信息等功能，帮助用户跟踪和分析镜像的使用情况。
 - 可扩展性和灵活性：Harbor具有良好的可扩展性，可以通过添加额外的Harbor节点来实现高可用性和负载均衡。它还提供了RESTful API和插件机制，可以与其他系统集成和扩展，满足用户的特定需求。

### 1. Harbor安装
#### 1.1 前提条件
a. Kubernetes集群版本>=1.20

b. Helm版本>=v3.2.0，其中 Helm 的安装请参考：[Helm Install](https://helm.sh/docs/intro/install/)

c.需要有默认的StorageClass，具体准备流程参考：[Kubernetes安装StorageClass](https://blog.csdn.net/weixin_46660849/article/details/130881771)

d.需要由默认的IngressClasses，具体准备流程参考：[Kubernetes安装IngressClass](https://blog.csdn.net/weixin_46660849/article/details/130904799) 其中第4步“设置为默认Ingress Class”是必须的，同时Ingress的NodePort端口最好设置为80、443
#### 1.2 安装流程
在集群的master节点进行以下操作：
a. 添加 Harbor Chart 仓库
```bash
helm repo add harbor https://helm.goharbor.io 
```
b. 创建安装Harbor的名字空间
```bash
kubectl create ns harbor
```
c. 执行Chart安装命令，服务暴露方式使用默认的Ingress

说明：由于有默认的 IngressClasses 以及 StorageClass，所以安装时具体的参数无需指定具体的 IngressClasses 以及 StorageClass，其中默认的tls加密证书也是流程自动生成的（当然也可以手动设置）。
具体参数的介绍参考：harbor chart官网
[harbor chart官网](https://artifacthub.io/packages/helm/harbor/harbor) 
相关流程也可以参考 （注意视频中的chart版本为v1.0.0，仅具有参考价值）： 
[参考视频](https://www.bilibili.com/video/BV14S4y1J7Td/?spm_id_from=333.337.search-card.all.click&vd_source=c29814fb5a844335f87ca53d3cfb0ac5) 
```bash
helm install harbor harbor/harbor \
--set externalURL=https://harbor.example.com \ #对外访问地址
--set expose.ingress.hosts.core=harbor.example.com \ #ingress.hosts.core地址，要和externalURL后的域名一致
--set expose.ingress.hosts.notary=notary.example.com \ #ingress.hosts.notary地址
--set harborAdminPassword=Yiqi123 \ #默认admin用户密码
-n harbor #安装的名字空间
```
d. 完成后可以通过如下命令查看是都安装成功
```bash
kubectl get po,ing,svc -n harbor
NAME                                        READY   STATUS    RESTARTS       AGE
pod/harbor-core-84dccff85b-7qlkd            1/1     Running   0              120m
pod/harbor-database-0                       1/1     Running   0              120m
pod/harbor-jobservice-f4689d655-4tqrc       1/1     Running   4 (119m ago)   120m
pod/harbor-notary-server-7d4b6ff68-xpjb5    1/1     Running   1 (119m ago)   120m
pod/harbor-notary-signer-665bc967c8-7x79d   1/1     Running   1 (119m ago)   120m
pod/harbor-portal-7d5f8d86cf-2qxl2          1/1     Running   0              120m
pod/harbor-redis-0                          1/1     Running   0              120m
pod/harbor-registry-75fcfd8b8c-qz4vg        2/2     Running   0              120m
pod/harbor-trivy-0                          1/1     Running   0              120m

NAME                                              CLASS   HOSTS                ADDRESS        PORTS     AGE
ingress.networking.k8s.io/harbor-ingress          nginx   harbor.example.com   172.16.80.22   80, 443   120m
ingress.networking.k8s.io/harbor-ingress-notary   nginx   notary.example.com   172.16.80.22   80, 443   120m

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/harbor-core            ClusterIP   172.16.226.24    <none>        80/TCP              120m
service/harbor-database        ClusterIP   172.16.138.139   <none>        5432/TCP            120m
service/harbor-jobservice      ClusterIP   172.16.90.83     <none>        80/TCP              120m
service/harbor-notary-server   ClusterIP   172.16.51.31     <none>        4443/TCP            120m
service/harbor-notary-signer   ClusterIP   172.16.238.7     <none>        7899/TCP            120m
service/harbor-portal          ClusterIP   172.16.178.86    <none>        80/TCP              120m
service/harbor-redis           ClusterIP   172.16.125.72    <none>        6379/TCP            120m
service/harbor-registry        ClusterIP   172.16.155.145   <none>        5000/TCP,8080/TCP   120m
service/harbor-trivy           ClusterIP   172.16.201.86    <none>        8080/TCP            120m
```
### 2. 访问验证
a. 在需要访问的机器设置hosts配置
```bash
vi /etc/hosts

#添加如下配置
<集群中任意Worker节点的Ip地址> harbor.example.com
```

b. 浏览器访问
![在这里插入图片描述](https://img-blog.csdnimg.cn/75bd150288e44de49e17a6c627b2d732.png)
c. 推送镜像设置，由于当前镜像仓库使用的tls证书为自签名的，所以是非授信仓库，需要在访问的docker配置文件中设置非授信仓库配置
```bash
vi /etc/docker/daemon.json
#添加如下内容：
{
  "insecure-registries": [
    "harbor.example.com"
  ]
}

#写完配置文件后执行以下命令：
systemctl daemon-reload
systemctl restart docker

#通过 docker login 登录私有仓库
docker login harbor.example.com

#镜像打标签
docker tar nginx:alpine harbor.example.com/library/nginx:alpine

#镜像推送
docker push harbor.example.com/library/nginx:alpine
```
完成推送后在图形化界面即可看到推送的镜像
![在这里插入图片描述](https://img-blog.csdnimg.cn/fe7f46e6722743d788c8edc8477b9ea7.png)
