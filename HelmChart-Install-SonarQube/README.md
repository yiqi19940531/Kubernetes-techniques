## Helm Chart Kubernetes安装SonarQube
SonarQube（经常简称为 Sonar）是一个开源平台，用于自动检测代码的质量，并提供详细的代码质量报告。它可以检测代码中的错误、漏洞和不合规的编码习惯，并提供一些有关如何解决这些问题的建议。SonarQube 支持多种编程语言，因此它适用于多种软件项目。

以下是 SonarQube 的一些关键特点和功能：
1. **多语言支持**：SonarQube 支持 20+ 的主要编程语言，包括 Java、C#、Python、JavaScript、TypeScript、Ruby、Go 等。

2. **代码质量度量**：SonarQube 提供了许多代码质量度量，包括但不限于代码复杂性、重复的代码、单元测试覆盖率、代码缺陷数量等。

3. **持续检查**：SonarQube 可以集成到 CI/CD 管道中，这样在每次代码更改时，都可以自动检查代码质量。

4. **可视化报告**：通过 SonarQube 的 Web 界面，开发者和团队可以轻松地访问详细的代码质量报告。

5. **安全性**：SonarQube 可以检测潜在的安全问题，如漏洞和不安全的编码实践，并将其分类为关键、重要、次要和信息四个等级。

6. **技术债务**：SonarQube 提供了一个"技术债务"的概念，帮助开发者估算解决代码质量问题需要的时间。

7. **扩展性**：通过插件，SonarQube 可以轻松地扩展，以支持更多的编程语言或增加新功能。

8. **与开发工具集成**：SonarQube 可以与常见的开发工具和平台集成，如 Jenkins、GitLab、GitHub Actions、Azure DevOps 等。

9. **代码品质门槛**：团队可以设置代码质量的标准或“门槛”，以确保新的代码更改不会降低整体的代码质量。

SonarQube也是CI/CD流水线部署中CI中的重要一环。
### 1.SonarQube
#### 1.1 前提条件
a. Helm版本>=v3.2.0，其中 Helm 的安装请参考：[Helm Install](https://helm.sh/docs/intro/install/)

b.需要有默认的StorageClass，具体准备流程参考：[Kubernetes安装StorageClass](https://blog.csdn.net/weixin_46660849/article/details/130881771)
#### 1.2 下载Chart模板
过程中用到SonarQube官方提供的chart模板，模板官网参考：[SonarQube Chart官方页面](https://artifacthub.io/packages/helm/sonarqube/sonarqube)

```bash
# 创建安装用文件夹
cd ~ && mkdir sonarqube

# 下载 sonarqube 官方
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube

helm pull sonarqube/sonarqube --version 10.2.1+800 #具体的版本取决于安装时的最新版本
```
#### 1.3 对values.yaml文件进行调整

```bash
tar -xvf sonarqube-10.2.1+800.tgz

cd sonarqube

vi values.yaml

# 全局搜索"service"关键字
service:
  type: NodePort  # 类型修改为NodePort
  externalPort: 9000
  internalPort: 9000
  nodePort: 30081 # NodePort对外暴露的端口
  labels:
  annotations: {}

# 如果需要使用ingress可以全局搜索"ingress"并修改其后先关参数

# 全局搜索"persistence"，找到第一处。修改持久化相关的配置
persistence:
  enabled: true  #设置为true
  ## Set annotations on pvc
  annotations: {}

  ## Specify an existing volume claim instead of creating a new one.
  ## When using this option all following options like storageClass, accessMode and size are ignored.
  # existingClaim:

  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  storageClass: nfs-client  #设置为当前集群默认的StorageClass
  accessMode: ReadWriteOnce
  size: 5Gi
  uid: 1000

  ## Specify extra volumes. Refer to ".spec.volumes" specification : https://kubernetes.io/fr/docs/concepts/storage/volumes/
  volumes: []
  ## Specify extra mounts. Refer to ".spec.containers.volumeMounts" specification : https://kubernetes.io/fr/docs/concepts/storage/volumes/
  mounts: []

# 全局搜索"persistence"，找到第二处。修改psql数据库相关的持久化设置
  persistence:
    enabled: true
    accessMode: ReadWriteOnce
    size: 20Gi
    storageClass: nfs-client #设置为当前集群默认的StorageClass
```
如需下载我提供的完整版Value.yaml文件请参考，[文件下载链接](https://download.csdn.net/download/weixin_46660849/88382756)
#### 1.4 安装

```bash
# 创建一个安装SonarQube用的名字空间 sonarqube
kubectl create ns sonarqube

# chart安装SonarQube
helm install sonarqube ./sonarqube -n sonarqube

# 安装后可以看到如下信息，填入后可以看到基于NodePort端口的访问方式
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace sonarqube -o jsonpath="{.spec.ports[0].nodePort}" services sonarqube-sonarqube)
  export NODE_IP=$(kubectl get nodes --namespace sonarqube -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
```
#### 1.5 页面登录
默认的登录用户名密码为：admin/admin
相关默认的密码也可以在values.yaml文件中设置（感兴趣可以搜索"adminPassword"关键字），但是不推荐，因为从页面登录后会要求用户重新设置admin账号登录的用户名密码。
![在这里插入图片描述](https://img-blog.csdnimg.cn/1c8f86b4c03747d8b99d83533c1a5ad2.png)
#### 1.6 设置中文插件（可选步骤）
进入Administration->Marketplace路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/a5cd1ced8c7843fa9601b32274545304.png)
在Plugins中搜索chinese，选择评分高的插件点击"Install"按钮，之后需要重启SonarQube服务。
![在这里插入图片描述](https://img-blog.csdnimg.cn/6a4a83effef8409392c9c48982f4e6bc.png)
重启完成后即可看到中文效果
![在这里插入图片描述](https://img-blog.csdnimg.cn/925d68c41dea45dd83336c957becbf1d.png)
