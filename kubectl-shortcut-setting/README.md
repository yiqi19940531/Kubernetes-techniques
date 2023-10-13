## 综述
kubectl（Kube Control）是Kubernetes的命令行工具，用于与Kubernetes集群进行交互和管理。kubectl这个命令由7个字母组成，手敲起来还是比较费劲的。如果为这个命令设置了简写以及命令自动补全，在配合“api-resources”本身提供的缩写，kubectl命令用起来就会丝滑很多。以下的设置在Centos机器中亲测有效且好用。
### 1. kubectl设置简写
```bash
cat >> ~/.bashrc << EOF 
alias kg='kubectl get'
alias k='kubectl'
alias kd='kubectl describe pods'
alias ke='kubectl explain'
alias ka='kubectl apply'
EOF

source ~/.bashrc
```
完成设置后
```bash
kg po #查看default名字空间下的po数量
NAME                                              READY   STATUS    RESTARTS       AGE
busybox                                           1/1     Running   1 (21d ago)    21d
counter                                           1/1     Running   5 (62d ago)    134d
http-svc-65b48ffc7b-2dzqt                         1/1     Running   6 (62d ago)    138d
http-svc-65b48ffc7b-b6rtz                         1/1     Running   5 (62d ago)    138d
http-svc-65b48ffc7b-sqtvt                         1/1     Running   5 (62d ago)    138d
nfs-subdir-external-provisioner-796df7c59-gcwc2   1/1     Running   39 (37d ago)   139d
nginx-demo-98c8d48ff-r6wmv                        1/1     Running   5 (62d ago)    156d

# 另外说一个技巧，可以通过"kubectl api-resource"查看当前集群中拥有的api-resources，其中第二个栏位“SHORTNAMES”对应相关资源的简写。
k api-resources  #实际输出内容较多，这里支截取了精华部分
NAME                              SHORTNAMES                                      APIVERSION                             NAMESPACED   KIND
bindings                                                                          v1                                     true         Binding
componentstatuses                 cs                                              v1                                     false        ComponentStatus
configmaps                        cm                                              v1                                     true         ConfigMap
endpoints                         ep                                              v1                                     true         Endpoints
events                            ev                                              v1                                     true         Event
limitranges                       limits                                          v1                                     true         LimitRange
namespaces                        ns                                              v1                                     false        Namespace
nodes                             no                                              v1                                     false        Node
persistentvolumeclaims            pvc                                             v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv                                              v1                                     false        PersistentVolume
pods                              po                                              v1                                     true         Pod
podtemplates                                                                      v1                                     true         PodTemplate
replicationcontrollers            rc                                              v1                                     true         ReplicationController
resourcequotas                    quota                                           v1                                     true         ResourceQuota
secrets                                                                           v1                                     true         Secret
serviceaccounts                   sa                                              v1                                     true         ServiceAccount
services                          svc                                             v1                                     true         Service

# 例如 "kg ns" 就等价于 “kubectl get namespace”
kg ns
NAME                STATUS   AGE
calico-apiserver    Active   156d
calico-system       Active   156d
cronjob             Active   131d
default             Active   156d
git                 Active   120d
har                 Active   136d
ingress-nginx       Active   138d
ingress-test        Active   138d
jenkins             Active   125d
kube-node-lease     Active   156d
kube-public         Active   156d
kube-system         Active   156d
liberty-app         Active   64d
logging             Active   134d
monitoring          Active   133d
ns-test             Active   138d
sonarqube           Active   113d
springboot-devops   Active   25d
sts                 Active   21d
tigera-operator     Active   156d
```
### 2. kubectl设置命令自动补全
1.安装bash-completion
```bash
yum install -y bash-completion 
source /usr/share/bash-completion/bash_completion
```
2.应用kubectl的completion到系统环境
```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
完成设置后

```bash
[root@yiqi-k8s-master1 ~]kube #按 “TAB”
kubeadm         kubectl         kubectl-calico  kubelet         
[root@yiqi-k8s-master1 ~]# kubectl #输入“kubec” 按 “TAB” 即可自动补全
```
