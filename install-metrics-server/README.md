# Kubernetes集群遇到“error: Metrics API not available”
导致这个问题原因是没有安装Metrics Server，接下来我会介绍Metrics Server在集群中的作用以及官方网站的解释。如果想直接解决报错可以看“解决“error: Metrics API not available”报错”章节。
### 1.Metrics Server作用理解
[Kubernetes官网链接](https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
Kubernetes Metrics Server是一个Kubernetes集群组件，它收集Kubernetes中各个对象（如节点、Pod、容器等）的度量指标数据，然后聚合和存储这些数据，以便其他Kubernetes组件和工具可以使用这些数据进行监控和调整。

Metrics Server采用了Kubernetes Metrics API的标准，它使用HTTP接口来暴露度量指标数据，并且通过轮询方式收集这些数据，然后将其存储在内存中。

通过Kubernetes Metrics API，可以获取各种度量指标数据，如CPU利用率、内存使用率、网络I/O、磁盘I/O等。这些指标可以帮助用户了解Kubernetes集群中各个对象的健康状态，帮助用户进行故障排查和性能优化。

Metrics Server对于Kubernetes集群的正常运行非常重要。如果Metrics Server未正常运行，将无法监测集群中各个对象的状态和性能，也将无法使用一些基于Metrics API的工具和组件，如Horizontal Pod Autoscaler (HPA)等。

在Kubernetes集群中，默认情况下并不包含Metrics Server组件，需要手动安装和配置。安装和配置Metrics Server的方法和流程因Kubernetes版本和环境而异。通常，可以使用Kubernetes官方提供的yaml文件进行安装，或者使用一些第三方的工具和插件进行安装和配置。

### 2.解决“error: Metrics API not available”报错
[官方解决方案](https://github.com/kubernetes-sigs/metrics-server)
官方的方案中包含了两种类型的Metrics Server安装
a.[常规安装单副本Metrics Server](https://github.com/kubernetes-sigs/metrics-server#installation)
b.[多副本高可用安装Metrics Server](https://github.com/kubernetes-sigs/metrics-server#high-availability)
以下介绍单副本安装方式
```bash
#下载yaml文件
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

#修改yaml文件中的部分内容
vi components.yaml
#需要修改的内容包括Deployment下spec.template.spec.containers.args内添加“- --kubelet-insecure-tls”
#Deployment下spec.template.spec.containers.image修改为“image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.6.3”

#直接定位到“containers”位点
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  #添加此行内容
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.6.3 #修改为国内镜像仓库地址
```
如果Kubernetes集群版本1.19+，可以直接使用以下yaml文件部署
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
rules:
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.6.3
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 4443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
```

完结撒花！！！
