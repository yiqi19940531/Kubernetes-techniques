Kubernetes 集群中创建一个包含 `kubectl` 命令的 Pod 通常用于管理和调试集群本身。这种 Pod 通常被称为“调试 Pod”或“管理 Pod”，它们的主要作用是允许从集群内部执行 Kubernetes 操作和管理任务。这可以在多种情况下非常有用：
### 集群管理和维护

- **内部访问**：在集群内部运行的 Pod 可以直接访问 Kubernetes API，这对于执行需要内部访问的管理任务非常方便。
- **权限隔离**：这些 Pod 可以配置有不同的权限，从而限制对集群的访问，确保安全。
### 调试和故障排除

- **内部视角调试**：当需要从集群内部的角度调试问题时（例如，网络问题或服务之间的通信问题），这些 Pod 可以提供必要的工具。
- **访问受限资源**：在某些情况下，这些 Pod 可以访问通常对外部客户端不可见的内部资源或服务。
### 自动化和作业

- **运行自动化任务**：这些 Pod 可以用于自动执行诸如备份、集群状态检查或自动化部署等任务。
- **作业和CronJobs**：在 Kubernetes 中执行定时任务或一次性作业，这些 Pod 可以作为执行这些任务的一部分。
### 实际案例
- **运行自动化任务**：之前的项目中就遇到过一种情况，某个名字空间下的Pod由于代码长时间运行会卡主（当然卡主的本质原因是代码质量问题），且Pod不会自动重启，这种情况下就可以通过具有cluster-admin权限的CronJobs容器定时重启容易出问题的Pod，这样就可以极大的降低Pod卡主的概率。
## Kubernetes集群中创建具有cluster-admin权限的容器
1.创建一个名字空间（创建名字空间的目的是为了更好的隔离资源）

```bash
kubeclt create ns kubectl-admin
```
2.创建ServiceAccount
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-service-account
  namespace: kubectl-admin
```
3.为ServiceAccount做RBAC权限绑定，我这里绑定的是cluster-admin的权限，当然实际情况下可以控制权限的绑定
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-role-binding
subjects:
- kind: ServiceAccount
  name: admin-service-account
  namespace: kubectl-admin
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
4.创建一个Deployment，运行容器
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubectl-deployment
  namespace: kubectl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubectl-app
  template:
    metadata:
      labels:
        app: kubectl-app
    spec:
      serviceAccountName: admin-service-account
      containers:
      - name: kubectl-container
        image: bitnami/kubectl
        command: ["tail", "-f", "/dev/null"]

```
5进入容器测试kubectl命令

```bash
kubectl exec -it kubectl-deployment-576894d47-m6rws -n kubectl-admin -- /bin/bash
I have no name!@kubectl-deployment-576894d47-m6rws:/$ kubectl get ns   
NAME               STATUS   AGE
cronjob            Active   58d
default            Active   183d
harbor             Active   50d
harbor-private     Active   57d
ingress-nginx      Active   50d
ingress-test       Active   50d
istio-system       Active   50d
kube-node-lease    Active   183d
kube-public        Active   183d
kube-system        Active   183d
kubectl            Active   47m
mysql              Active   4d9h
nginx-deployment   Active   116d
pg                 Active   16d
pgg                Active   10d
probe-test         Active   116d
rook-ceph          Active   41d
sonarqube          Active   84d
I have no name!@kubectl-deployment-576894d47-m6rws:/$ 
```
