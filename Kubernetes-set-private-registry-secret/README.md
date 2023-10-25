## Kubernetes拉取Pod失败
该错误消息 "401 Unauthorized" 指示尝试从指定镜像仓库拉取镜像时遇到了身份验证问题。
如果查看Pod的状态会看到：
```bash
kubectl get pod -n harbor-private
NAME                                         READY   STATUS             RESTARTS   AGE
harbor-private-deployment-5f98bbd88b-wf4bm   0/1     ImagePullBackOff   0          49m
```
如果进一步查看Pod Event信息可以看到"pulling from host harbor.example.com failed with status code [manifests v1]: 401 Unauthorized:" 这个错误明确提示了无法拉取镜像的原因是私有镜像仓库无法得到认证。
```bash
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  36s                default-scheduler  Successfully assigned harbor-private/harbor-private-deployment-5f98bbd88b-wf4bm to w2
  Normal   Pulling    20s (x2 over 36s)  kubelet            Pulling image "harbor.example.com/yiqi/nginx:v1"
  Warning  Failed     19s (x2 over 36s)  kubelet            Failed to pull image "harbor.example.com/yiqi/nginx:v1": rpc error: code = Unknown desc = failed to pull and unpack image "harbor.example.com/yiqi/nginx:v1": failed to resolve reference "harbor.example.com/yiqi/nginx:v1": pulling from host harbor.example.com failed with status code [manifests v1]: 401 Unauthorized
  Warning  Failed     19s (x2 over 36s)  kubelet            Error: ErrImagePull
  Normal   BackOff    6s (x3 over 35s)   kubelet            Back-off pulling image "harbor.example.com/yiqi/nginx:v1"
  Warning  Failed     6s (x3 over 35s)   kubelet            Error: ImagePullBackOff
```
### 方案1：通过创建docker-registry认证信息的Secret解决上述问题
1.使用 kubectl 命令来为 Docker registry 创建一个 Secret:
```bash
kubectl create secret docker-registry <secret-name> \
    --docker-server=DOCKER_REGISTRY_SERVER \
    --docker-username=DOCKER_USER \
    --docker-password=DOCKER_PASSWORD \
    --docker-email=DOCKER_EMAIL  #如果没有邮箱地址则随便填一个邮箱即可。
```
2.在你的 Deployment 资源定义中，你需要在 spec 下的每个容器中指定 imagePullSecrets 来引用上面创建的 Secret。例如：
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: harbor-private-deployment
  namespace: harbor-private
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: harbor-private
    spec:
      containers:
      - name: harbor-private-container
        image: harbor.example.com/yiqi/nginx:v1
      imagePullSecrets:
      - name: <secret-name>   #指定之前创建的secret名称
  selector:
    matchLabels:
      app: harbor-private
```
### 方案2：修改当前名字空间默认的ServiceAccount为其添加imagePullSecrets
1.使用 kubectl 命令来为 Docker registry 创建一个 Secret:
```bash
kubectl create secret docker-registry <secret-name> \
    --docker-server=DOCKER_REGISTRY_SERVER \
    --docker-username=DOCKER_USER \
    --docker-password=DOCKER_PASSWORD \
    --docker-email=DOCKER_EMAIL  #如果没有邮箱地址则随便填一个邮箱即可。
```
2.可以设置 ServiceAccount，使其默认为 namespace 中的所有 Pods 使用此 Secret，这样当你创建新的 Pod 时，不必为每个 Pod 单独指定 imagePullSecrets。
首先，编辑 ServiceAccount：
```bash
kubectl edit serviceaccount default -n <namesapce-name>

#然后，在 ServiceAccount 的定义下添加 imagePullSecrets 字段
imagePullSecrets:
- name: <secret-name>
#保存并退出
```
对于已经在 namespace 中运行的 Pods，你需要重新创建它们以使用新的 imagePullSecret。你可以先删除这些Pod，然后再重新创建它们，或者更新相关的 Deployment、StatefulSet 等控制器，这样新的 Pods 会自动创建并使用新的 Secret。

之后，在该 namespace 中创建的所有新 Pods 都将默认使用此 imagePullSecret，允许它们从私有仓库拉取镜像。已经存在的 Pods 需要重新创建才能使用新的 imagePullSecret。
