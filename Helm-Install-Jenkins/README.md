## Kubernetes集群Helm Chart安装Jenkins并设置简体中文
本文介绍如何通过Helm Chart方式快速在Kubernetes环境中安装Jenkins，并通过Ingress对服务进行对外暴露。还会介绍如何安装Jenkins汉化插件。
### Jenkins的作用
Jenkins是一个开源的持续集成/持续部署（CI/CD）工具，用于自动化各种任务，包括构建，测试和部署软件。Jenkins 是一个帮助自动化软件开发过程中的构建，测试，部署等重复任务的重要工具，它可以提高工作效率，减少人为错误，加快开发周期，最终提高软件的质量和交付速度。Jenkins主要用于开发过程中的以下几个方面：

 -  **持续集成（Continuous Integration）**：Jenkins 可以监控你的版本控制系统中的变更，自动编译和测试代码，使开发者能够快速发现和解决问题。
 - **持续部署（Continuous Delivery/Deployment）**：Jenkins还可以自动部署软件到生产环境，或者预设的其他环境。这使得交付更新和新版本的过程更加快速和容易。
 - **测试和报告**：Jenkins可以执行各种类型的测试，并产生详细的测试报告。这些报告可以在 Jenkins 的 Web 界面上查看。
 -  **分布式构建**：Jenkins 可以在多台机器上分布工作，让更大的项目可以在更短的时间内完成构建和测试。
 - **插件系统**：Jenkins 有一个庞大的插件生态系统，这使得 Jenkins 可以通过安装插件以满足几乎任何 CI/CD 场景的需求。
### 1. Helm Chart 安装 Jenkins 2.332.3
#### 1.1 前提条件
a. Helm版本>=v3.2.0，其中 Helm 的安装请参考：[Helm Install](https://helm.sh/docs/intro/install/)

b.需要有默认的StorageClass，具体准备流程参考：[Kubernetes安装StorageClass](https://blog.csdn.net/weixin_46660849/article/details/130881771)

c.有默认的IngressClasses，具体准备流程参考：[Kubernetes安装IngressClass](https://blog.csdn.net/weixin_46660849/article/details/130904799) 其中第4步“设置为默认Ingress Class”是必须的，同时Ingress的NodePort端口最好设置为80、443
#### 1.2 安装流程
a.创建安装目录
```bash
#切换到当前用户根目录并建立jenkins文件夹
cd ~ && mkdir jenkins

cd jenkins
```
b.创建jenkins名字空间，独立的名字空间有助于资源管理
```bash
kubectl create ns jenkins
```
c.添加jenkins官方repo仓库
```bash
helm repo add jenkinsci https://charts.jenkins.io/
```
d.安装jenkins
如果需要做某些定制化需求请参考官方 values.yaml 文件，其中每个参数都会有详细描述: [官网参考](https://artifacthub.io/packages/helm/jenkinsci/jenkins/4.1.3) 
```bash
helm pull jenkinsci/jenkins --version 4.1.3

tar -xvf jenkins-4.1.3.tgz

cd jenkins

# 我主要对Jenkins的初始化密码、服务暴露方式设置为Ingress、数据持久化绑定StorageClass。
vi values.yaml
#初始化密码，搜索 "adminPassword" 关键字(不设置，系统会生成一个默认的，通过Pod可以查看)
adminPassword: "<your password>"

#服务暴露方式设置为Ingress，搜索 "ingress:" 关键字，设置为如下
  ingress:
    enabled: true #开启
    paths: []
    # For Kubernetes v1.14+, use 'networking.k8s.io/v1beta1'
    # For Kubernetes v1.19+, use 'networking.k8s.io/v1'
    apiVersion: "networking.k8s.io/v1"
    labels: {}
    annotations: {}
    ingressClassName: nginx #Ingress的名称
    hostName: jenkins.yiqi.com
    #如果需要可以设置tls，这样即可通过https登录是就会有相应证书。由于我是自己测试，ingress后续会和很多系统做集成，使用非授信的ssl证书反而会带来很多麻烦。
    # tls:
    # - secretName: jenkins.cluster.local
    #   hosts:
    #     - jenkins.cluster.local

#数据持久化绑定StorageClass，搜索 "persistence:" 关键字，设置为如下
  enabled: true
  existingClaim:
  storageClass: "nfs-client" #设置为StorageClass名称
  annotations: {}
  labels: {}
  accessMode: "ReadWriteOnce"
  size: "8Gi"
  volumes:
  mounts:
```
e.执行安装命令
```bash
cd ~/jenkins

helm install jenkins ./jenkins -n jenkins

#成功后看到如下提示
NAME: jenkins
LAST DEPLOYED: Thu Jun 15 08:47:45 2023
NAMESPACE: jenkins
STATUS: deployed
REVISION: 2
NOTES:
1. Get your 'admin' user password by running: #查看默认admin用户密码
  kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo

2. Visit http://jenkins.yiqi.com

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://jenkins.yiqi.com/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins

#通过如下命令检测&确保Pod正确运行，容器会经历两个init阶段，故启动需要一定时间
kubectl get po -n jenkins -w
NAME        READY   STATUS    RESTARTS      AGE
jenkins-0   2/2     Running   2 (15h ago)   15h
```
### 2. 登录图形化界面并设置中文
#### 2.1 设置中文界面
a. 登录进入主页
![在这里插入图片描述](https://img-blog.csdnimg.cn/a84997fa83e3474f94ca7976b49b0014.png)
b. 点击"Manage Jenkins"
![在这里插入图片描述](https://img-blog.csdnimg.cn/672b13f0123448cd9af8d89290c3851a.png)
c. 在"Available"下搜索"Chinese"关键字，安装时勾选"Restart"选择框，完成后重启服务，再次登录就是中文界面。
![在这里插入图片描述](https://img-blog.csdnimg.cn/f565ecc12ef74ca29ed99a2ba2b7e56c.png)
