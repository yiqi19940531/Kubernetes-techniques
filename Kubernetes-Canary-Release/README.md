## Kubernetes集群通过StatefulSet实现应用金丝雀发布
金丝雀发布（Canary Release）的名字来源于“金丝雀在煤矿”的历史实践。在20世纪初，矿工们带着金丝雀进入煤矿，用它们作为一种早期的有毒气体警告系统。金丝雀对有毒气体，如一氧化碳的反应比人类更敏感。当金丝雀表现出中毒的迹象时，矿工们知道环境不安全并迅速撤离。这种方法为矿工提供了一个早期的警告，有助于保护他们的生命。

引申到软件发布，金丝雀发布意味着首先只向一个小部分用户推出新版本的应用或功能，而不是一次性发布到所有用户。这样，组织可以在全面部署之前观察新版本在实际环境中的表现，确保其稳定性和功能性。
### 金丝雀发布的优点
**降低风险**：通过逐渐发布新版本，你可以减少新代码引入的风险。如果新版本存在问题，只有一小部分用户会受到影响，而不是全部。

**快速反馈**：部署到一小部分用户后，你可以迅速收集反馈，了解新功能或更改是否达到预期效果。

**性能监控**：在部署到所有用户之前，你有机会观察新版本如何影响系统的性能。这可能包括服务器负载、响应时间、数据库性能等。

**更好的故障排除**：如果新版本存在问题，处理较小的用户群体比处理整个用户基数更为简单。这也使得回滚（如果需要的话）变得更加简单和快速。

**灵活的目标群体选择**：你可以基于特定的标准选择目标群体，例如地理位置、用户类型或其他属性，这样可以确保你的新功能首先针对最相关的用户群体。

**增加用户的信心**：知道组织采用了逐渐部署的策略，而不是盲目部署，可以增加用户对产品的信心。

**测试真实环境**：金丝雀发布提供了在真实环境中而不是仅在测试环境中测试新代码的机会。

总的来说，金丝雀发布是一个策略，通过将新版本逐渐推出，组织可以更安全、更有信心地发布新功能和更改，同时还可以获得及时的反馈和监控。
### Kubernetes中StatefulSet介绍
`StatefulSet` 是 Kubernetes 的一个工作负载API对象，它为部署和扩展一组有状态的应用提供了管理功能。与 `Deployment` 和 `ReplicaSet` 等无状态的工作负载资源相比，`StatefulSet` 主要用于应用程序需要一个或多个以下特性的情况：

**稳定且唯一的网络标识符**：每个 `StatefulSet` 中的 Pod 都有一个稳定的主机名，即使 Pod 被重新调度到其他节点也不会改变。这种主机名格式通常为 `$(statefulset-name)-$(ordinal-index)`。

 **稳定且持久的存储**：`StatefulSet` 可以与 `PersistentVolume` 配合使用，以确保存储持久化并与 Pod 的生命周期分离。当 Pod 被重新调度或删除时，关联的 PersistentVolume 不会丢失。

**有序部署和扩展**：当启动或扩展 `StatefulSet` 时，Pods 会按照数字顺序一次启动。同样地，当 Pods 被终止时，它们会按照相反的顺序关闭。

 **有序的滚动更新**：当你更新 `StatefulSet` 中的 Pods 的配置时，它们会按照相同的顺序逐一更新，而不是同时更新。

### StatefulSet的partition特性
`partition` 是一个特定于 `StatefulSet` 的属性，用于支持有序的滚动回滚。

当你对 `StatefulSet` 进行更新时，如修改其 Pod 模板，Kubernetes 会根据更新策略进行滚动更新，以更新 `StatefulSet` 中的 Pods。如果你使用的更新策略是 `RollingUpdate`（这是默认策略），则可以使用 `partition` 属性指定一个序号。这个序号表示，只有此序号之后的 Pods 会被更新，而之前的 Pods 会保持不变。

这种设置非常有用，尤其是在你希望逐步、手动地应用更新时。通过调整 `partition` 的值，你可以控制更新的速度和范围，以及随时回滚到先前的版本。

正应为有了 `partition`特性，可以用来实现应用的金丝雀发布。例如，如果你有一个有5个副本的 `StatefulSet`（从 `0` 到 `4` 的序号），并且你设置 `partition` 为 `3`，那么序号为 `3`、  `4`的 Pod 会收到更新。序号为 `0`、`1` 和 `2` 的 Pods 将不会被更新，除非你进一步更改 `partition` 的值。
### Kubernetes集群通过StatefulSet金丝雀发布实战部署
#### 1.1 前提条件
a. 需要有安装好的Kubernetes集群，可以参考：[Kubeadm安装K8s1.26集群](https://blog.csdn.net/weixin_46660849/article/details/130580689)

b.需要有一个镜像仓库，具体准备流程参考：[Kubernetes安装Harbor](https://blog.csdn.net/weixin_46660849/article/details/130934077)

c.有了镜像仓库后很可能需要设置镜像仓库的非授信访问：[Linux机器配置信任Harbor自签名ssl证书](https://blog.csdn.net/weixin_46660849/article/details/132550969) 
#### 1.2 制作两个版本的Nginx镜像
Nginx容器可以直接通过浏览器访问网页看到实际部署上去的效果非常直观，因此只做两个版本的Nginx镜像。
```bash
# 创建一个新目录，用于存放docker镜像相关的文件
cd ~ && mkdir nginx-image

# 创建Dockerfile
cat <<EOL > Dockerfile
# 使用官方的 nginx 基础镜像
FROM nginx:latest
# 复制 index.html 到 nginx 的默认目录
COPY version.html /usr/share/nginx/html/
EOL

# 创建V1版本的version.html页面（其实就是在HTML页面标注当前的页面是V1）
cat <<EOL > version.html
<!DOCTYPE html>
<html>
<head>
<title>Version Page</title>
</head>
<body>
<h1>This is V1</h1>
</body>
</html>
EOL

# 制作V1版本的Nginx镜像
docker build -t version-test:v1 .

# 创建V2版本的version.html页面（其实就是在HTML页面标注当前的页面是V2）
rm -f version.html # 先删除原来V1版本的页面

cat <<EOL > version.html
<!DOCTYPE html>
<html>
<head>
<title>Version Page</title>
</head>
<body>
<h1>This is V2</h1>
</body>
</html>
EOL

# 制作V2版本的Nginx镜像
docker build -t version-test:v2 .
```
完成上述步骤后查看本地镜像仓库

```bash
docker images
REPOSITORY                            	TAG                             	IMAGE ID   	CREATED    	SIZE
version-test                          	v2                              	fushvbruv78h   2 hours ago	187MB
version-test                          	v1                              	efmcsiof1295   2 hours ago	187MB
```
#### 1.3 镜像推送到Harbor仓库
```bash
# 镜像打标签
docker tag version-test:v1 harbor.example.com/library/version-test:v1
docker tag version-test:v2 harbor.example.com/library/version-test:v2

# 镜像推送Harbor仓库
docker push harbor.example.com/library/version-test:v1
docker push harbor.example.com/library/version-test:v2
```
#### 1.4 完成V1版本的部署测试

```bash
# 创建一个名为sts的名字空间
kubectl create ns sts

# 完成V1版本的部署
cat <<EOL > deploy.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
  namespace: sts
spec:
  serviceName: "nginx"
  replicas: 5     #副本数设置5便于观察
  selector:
	matchLabels:
  	app: nginx
  template:
	metadata:
  	labels:
    	app: nginx
	spec:
  	containers:
  	- name: nginx
    	image: harbor.example.com/library/version-test:v1     #先部署V1版本的镜像
    	ports:
    	- containerPort: 80
      	name: web
  updateStrategy:
	rollingUpdate:
  	partition: 0     #partition设置为0，代表从代号0开始的Pod
	type: RollingUpdate
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
	app: nginx
  namespace: sts
spec:
  type: NodePort
  ports:
  - port: 80
	targetPort: 80
	name: web
	nodePort: 36080
  selector:
	app: nginx
EOL

# 部署
kubectl apply -f deploy.yaml

# 检查Pod
kubectl get po -n sts
NAME              	READY   STATUS	RESTARTS   AGE
nginx-statefulset-0   1/1 	Running   0      	7m8s
nginx-statefulset-1   1/1 	Running   0      	7m6s
nginx-statefulset-2   1/1 	Running   0      	7m5s
nginx-statefulset-3   1/1 	Running   0      	7m3s
nginx-statefulset-4   1/1 	Running   0      	7m1s

# 检查Pod镜像版本，可以看到5个副本的镜像版本都是V1
kubectl get pods -n sts -o=jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}'
harbor.example.com/library/version-test:v1
harbor.example.com/library/version-test:v1
harbor.example.com/library/version-test:v1
harbor.example.com/library/version-test:v1
harbor.example.com/library/version-test:v1
```
#### 1.5 完成V1版本与V2版本之间的金丝雀部署
现在的目标是将其中2个Pod升级为V2版本，其余的Pod仍然保持V1版本

```bash
# 修改deploy.yaml文件，spec.template.containers.image修改为v2版本镜像，spec.updateStrategy.partition修改为3
image: harbor.example.com/library/version-test:v2    #v2版本镜像

partition: 3  #partition修改为3

# 更新部署
kubectl apply -f deploy.yaml
statefulset.apps/nginx-statefulset configured
service/nginx unchanged

# 查看pod状态，可以看到后缀为“-3”、“-4”的容器已经被修改
kubectl get po -n sts
NAME              	READY   STATUS	RESTARTS   AGE
nginx-statefulset-0   1/1 	Running   0      	17m
nginx-statefulset-1   1/1 	Running   0      	17m
nginx-statefulset-2   1/1 	Running   0      	17m
nginx-statefulset-3   1/1 	Running   0      	20s
nginx-statefulset-4   1/1 	Running   0      	23s

# 查看Pod对应的Image版本
kubectl get pods -n sts -o=jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}'
harbor.example.com/library/version-test:v1
harbor.example.com/library/version-test:v1
harbor.example.com/library/version-test:v1
harbor.example.com/library/version-test:v2
harbor.example.com/library/version-test:v2
```
#### 1.6 页面访问验证
从访问的页面上也可以看出会存在两个不同的版本。
![在这里插入图片描述](https://img-blog.csdnimg.cn/aaef580c367c471497d7ed46bf35cf7a.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/1e08d3bf8ba347c7ba5d4d2b06a07a5f.png)
