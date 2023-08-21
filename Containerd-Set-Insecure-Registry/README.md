## containerd 介绍
containerd 是一个独立的容器运行时，用于执行和管理容器。它是为稳定性、性能和可移植性而设计的，而不是一个完整的容器解决方案，而是提供了必要的核心功能来运行容器。containerd 与 Open Container Initiative (OCI) 标准兼容，这意味着它可以与任何 OCI 兼容的容器映像和运行时一起工作。
Kubernetes 在 1.24 版本里弃用并移除 dockershim。使用 Docker 引擎作为其 Kubernetes 集群的容器运行时的工作流或系统需要在升级到 1.24 版本之前进行迁移。如此以来 containerd 在 Kubernetes 集群中使用的更为广泛。
### containerd 未设置非授信仓库拉取私有仓库报错
可以看到在私有的 Harbor 仓库中有名为 myredis:latest 的镜像。如果对 Kubernetes 中如何安装 Harbor 仓库感兴趣的可以查看 [Kubernetes通过Helm Chart安装Harbor仓库并访问验证](https://blog.csdn.net/weixin_46660849/article/details/130934077)
![在这里插入图片描述](https://img-blog.csdnimg.cn/45b8eb2043444908b3aec4a13e4e30fd.png)
如果我们通过 crictl 命令拉取镜像时会遇到如下错误提示：

```bash
crictl pull harbor.example.com/library/myredis:latest
E0821 05:02:17.961874   16380 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"harbor.example.com/library/myredis:latest\": failed to resolve reference \"harbor.example.com/library/myredis:latest\": failed to do request: Head \"https://harbor.example.com/v2/library/myredis/manifests/latest\": x509: certificate signed by unknown authority" image="harbor.example.com/library/myredis:latest"
FATA[0000] pulling image: rpc error: code = Unknown desc = failed to pull and unpack image "harbor.example.com/library/myredis:latest": failed to resolve reference "harbor.example.com/library/myredis:latest": failed to do request: Head "https://harbor.example.com/v2/library/myredis/manifests/latest": x509: certificate signed by unknown authority 
```
看到 x509: certificate signed by unknown authority 就可以想到是由于拉取的镜像来源于非授信仓库。
### containerd 设置非授信仓库

```bash
vi /etc/containerd/config.toml

# 搜索有关registry设置的关键字 "plugins."io.containerd.grpc.v1.cri".registry]"

#将 "plugins."io.containerd.grpc.v1.cri".registry.configs" 下的内容设置为：
[plugins."io.containerd.grpc.v1.cri".registry.configs]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.example.com".tls]
    insecure_skip_verify = true

#将 "plugins."io.containerd.grpc.v1.cri".registry.mirrors" 下的内容设置为：
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor.example.com"]
    endpoint = ["https://harbor.example.com"]
```
保存退出后重启 containerd

```bash
sudo systemctl restart containerd
```
之后再次拉取镜像就不会出现 x509 相关的错误提示

```bash
crictl pull harbor.example.com/library/myredis:latest
Image is up to date for sha256:506734eb5e71d5555e75ecfe2253806f75a6845cec83d7e03ca974137e9dc622
```
