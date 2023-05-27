## Kubernetes Ingress Controller 介绍&安装&测试
### 基本介绍
对于Kubernetes的Service，无论是Cluster-Ip和NodePort均是4层的负载，集群内的服务如何实现七层的负载均衡。Ingress-nginx是7层的负载均衡器 ，负责统一管理外部对k8s cluster中Service的请求。

接下来第一个章节会介绍当前主流的Ingress Controller实现方案。如果想直接了解 Kubernetes Ingress Controller 请从第二章节开始食用。

### 1. Ingress Nginx选择
本章内容 [参考博客](https://juejin.cn/post/7051131623047168037)
目前Ingress是暴露集群内服务的行内公认最好的方式，不过由于其重要地位，世面上有非常多的Ingres Controller，常见的有：
 - Kubernetes Ingress
 - Nginx Ingress
 - Kong Ingress
 - Traefik Ingress
 - HAProxy Ingress
 - Istio Ingress
 - APISIX Ingress
 
除了上面列举的这些，还有非常多的Ingress Controller，面对如此多的Ingress Controller，我们该如何选择呢？参考的标准是什么？

一般情况下可以从以下几个维度进行判断：

 - 支持的协议：是否支持除HTTP(S)之外的协议
 - 路由的规则：有哪些转发规则，是否支持正则
 - 部署策略：是否支持ab部署、金丝雀部署、蓝绿部署等
 - upstream探针：通过什么机制判定应用程序正常与否，是否有主动和被动检查，重试，熔断器，自定义运行状况检查等解决方案
 - 负载均衡算法：支持哪些负载均衡算法，Hash、会话保持、RR、WRR等
 - 鉴权方式：支持哪些授权方案？基本，摘要，OAuth，外部身份验证等
 - DDoS防护能力：是否支持基本的限速、白名单等
 - 全链路跟踪：能否正常接入全链路监控
 - 全链路跟踪：能否正常接入全链路监控
 - JWT验证：是否有内置的JSON Web令牌验证，用于对最终应用程序的用户进行验证和验证
 - 图像界面：是否需要图形界面
 - 定制扩展性：是否方便扩展

下面分别对上述的Ingress Controller做简要介绍。

#### 1.1 Kubernetes Ingress
[官方仓库](https://github.com/kubernetes/ingress-nginx)
Kubernetes Ingress的官方推荐的Ingress控制器，它基于nginx Web服务器，并补充了一组用于实现额外功能的Lua插件。

由于Nginx的普及使用，在将应用迁移到K8S后，该Ingress控制器是最容易上手的控制器，而且学习成本相对较低，如果你对控制器的能力要求不高，建议使用。

不过当配置文件太多的时候，Reload是很慢的，而且虽然可用插件很多，但插件扩展能力非常弱。
#### 1.2 Nginx Ingress
[官方仓库](https://github.com/nginxinc/kubernetes-ingress)
Nginx Ingress是NGINX开发的官方版本，它基于NGINX Plus商业版本，NGINX控制器具有很高的稳定性，持续的向后兼容性，没有任何第三方模块，并且由于消除了Lua代码而保证了较高的速度（与官方控制器相比）。

相比官方控制器，它支持TCP/UDP的流量转发，付费版有很广泛的附加功能，主要缺点就是缺失了鉴权方式、流量调度等其他功能。
#### 1.3 Kong Ingress
[官方仓库](https://github.com/Kong/kubernetes-ingress-controller)
Kong Ingress建立在NGINX之上，并增加了扩展其功能的Lua模块。

kong在之前是专注于API网关，现在已经成为了成熟的Ingress控制器，相较于官方控制器，在路由匹配规则、upstream探针、鉴权上做了提升，并且支持大量的模块插件，并且便与配置。

它提供了一些 API、服务的定义，可以抽象成 Kubernetes 的 CRD，通过Kubernetes Ingress 配置便可完成同步状态至 Kong 集群。
#### 1.4 Traefik Ingress
[官方仓库](https://github.com/traefik/traefik)
traefik Ingress是一个功能很全面的Ingress，官方称其为：Traefik is an  Edge Router that makes publishing your services a fun and easy experience.

它具有许多有用的功能：连续更新配置（不重新启动），支持多种负载平衡算法，Web UI，指标导出，支持各种协议，REST API，Canary版本等。开箱即用的“Let's Encrypt”支持是另一个不错的功能。而且在2.0版本已经支持了TCP / SSL，金丝雀部署和流量镜像/阴影等功能，社区非常活跃。
#### 1.5 Istio Ingress
[官方仓库](https://istio.io/latest/docs/tasks/traffic-management/ingress/)
Istio是IBM，Google和Lyft（Envoy的原始作者）的联合项目，它是一个全面的服务网格解决方案。它不仅可以管理所有传入的外部流量（作为Ingress控制器），还可以控制集群内部的所有流量。在幕后，Istio将Envoy用作每种服务的辅助代理。从本质上讲，它是一个可以执行几乎所有操作的大型处理器。其中心思想是最大程度的控制，可扩展性，安全性和透明性。

借助Istio Ingress，您可以微调流量路由，服务之间的访问授权，平衡，监控，金丝雀发布等。

不过社区现在更推荐使用Ingress Gateways。
#### 1.6 HAProxy Ingress
[官方仓库](https://github.com/jcmoraisjr/haproxy-ingress)
HAProxy作为王牌的负载均衡器，在众多控制器中最大的优势还在负载均衡上。

它提供了“软”配置更新（无流量丢失），基于DNS的服务发现，通过API的动态配置。HAProxy还支持完全自定义配置文件模板（通过替换ConfigMap）以及在其中使用Spring Boot函数。
#### 1.7 APISIX Ingress
[官方仓库](https://github.com/apache/apisix-ingress-controller)
ApiSix Ingress是一个新兴的Ingress Controller，它主要对标Kong Ingress。

它具有非常强大的路由能力、灵活的插件拓展能力，在性能上表现也非常优秀。同时，它的缺点也非常明显，尽管APISIX开源后有非常多的功能，但是缺少落地案例，没有相关的文档指引大家如何使用这些功能。

附一张对比图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/26fb38b0eb8d485794db19d30545963d.webp)
### 2. Kubernetes Ingress Controller
本节介绍如何安装 Kubernetes Ingress Controller。我会采用NodePort的端口暴露方式，将 ingress-nginx-controller service 以 NodePort 方式暴露80,443端口。
#### 2.1 Kubernetes 开放1-65535 NodePort端口
如果是用 kubeadm 安装的集群，那么 apiserver 是以静态 pod 的形式运行，pod 文件定义在 /etc/kubernetes/manifests/kube-apiserver.yaml。/etc/kubernetes/manifests 目录下是所有静态 pod 文件的定义，kubelet 会监控该目录下文件的变动，只要发生变化，pod 就会重建，响应相应的改动。所以我们修改 /etc/kubernetes/manifests/kube-apiserver.yaml 文件，添加 nodePort 范围参数后会自动生效。
```bash
#编辑apiserver静态容器yaml文件
vi /etc/kubernetes/manifests/kube-apiserver.yaml

#在spec.containers.command下添加
- --service-node-port-range=1-65535

#完成后重启apiserver容器
kubectl delete pods -n kube-system -l component=kube-apiserver
```
#### 2.2 安装Kubernetes Ingress Nginx
[官网指南](https://kubernetes.github.io/ingress-nginx/deploy/)
这里采用 YAML manifest 文件的安装方式，安装前注意Kubernetes版本和 Kubernetes Ingress Nginx 之间的版本对应：
![在这里插入图片描述](https://img-blog.csdnimg.cn/f0a9055b009c4cf69d67326f69ee5161.png)
```bash
#切换到当前用户目录
cd ~

mkdir ingress-nginx

#下载安装用 manifest 文件
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.1/deploy/static/provider/cloud/deploy.yaml
```
编辑 deploy.yaml 修改安装方式为 NodePort 并指定特定端口
```bash
vi deploy.yaml 

#找到 ingress-nginx-controller 对应的service，可以通过在vi模式下搜索“LoadBalancer”关键字
修改为如下内容：
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.7.1
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Local
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    nodePort: 80 #http开放80作为NodePort端口
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    nodePort: 443 #https开放443作为NodePort端口
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: NodePort #服务方式设置为NodePort
```
用修改后的deploy.yaml执行安装
```bash
kubectl apply -f deploy.yaml

#检查是否安装成功，看到一下状态则表示安装成功
[root@k8s-master ingress-controller]# kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-4cjmw        0/1     Completed   0          8h
ingress-nginx-admission-patch-5lqbc         0/1     Completed   1          8h
ingress-nginx-controller-6599b4f4c5-rkjlz   1/1     Running     0          7h51m

#查看相关的service
[root@k8s-master ingress-controller]# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                 AGE
ingress-nginx-controller             NodePort    172.16.80.22    <none>        80:80/TCP,443:443/TCP   8h
ingress-nginx-controller-admission   ClusterIP   172.16.85.159   <none>        443/TCP                 8h
```
### 3. 功能验证
具体分为以下内容
1.创建名为 ingress-test 的名字空间，用于测试ingress
2.创建一个deployment，运行nginx服务，添加标签：app=nginx
3.创建一个service，端口为80，并配置selector为app=nginx，用于指向nginx pod
4.创建一个ingress，设置service为name=nginx，port=80，用于指向nginx service，并配置域名test.nginx.com
```bash
#创建名为 test-nginx.yaml 的文件
vi test-nginx.yaml

#文件中添加如下内容
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: ingress-test
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: ingress-test
spec:
  ports:
    - port: 80
      targetPort: 80
      name: nginx
  selector:
    app: nginx
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: ingress-test
  annotations: 
    kubernetes.io/ingress.class: "nginx"    # 指定 Ingress Controller 的类型
    nginx.ingress.kubernetes.io/use-regex: "true"    # 指定我们的 rules 的 path 可以使用正则表达式
spec:
  rules:
    - host: test.nginx.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```
完成后部署上述文件，并检测相关内容
```bash
kubectl apply -f test-nginx.yaml

#检查相关资源部署情况
[root@k8s-master ingress-controller]# kubectl get po,svc,ing -n ingress-test
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-85996f8dbd-5t45h   1/1     Running   0          39s
pod/nginx-85996f8dbd-srs7b   1/1     Running   0          39s

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/nginx   ClusterIP   172.16.144.245   <none>        80/TCP    40s

NAME                                   CLASS    HOSTS            ADDRESS        PORTS   AGE
ingress.networking.k8s.io/nginx-test   <none>   test.nginx.com   172.16.80.22   80      39s
```
通过本机的IP访问部署的网页
```bash
#此时 curl —H 以及本机的ip后可以看到如下效果
[root@k8s-master ingress-controller]# curl -H "Host: test.nginx.com" http://<本机ip地址>
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
