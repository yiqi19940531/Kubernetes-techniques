## Linux机器配置信任Harbor自签名ssl证书
Harbor 是一个流行的开源容器镜像仓库，它支持容器镜像的存储、签名、扫描和复制功能。为了确保与 Harbor 的通信是安全的，通常会为其配置 SSL/TLS。然而，为了避免购买证书或简化部署过程，很多人在部署 Harbor 时选择使用自签名证书。

配置 Linux 机器以信任 Harbor 的自签名 SSL 证书有以下好处：
1. **安全通信**：SSL/TLS 证书确保与 Harbor 服务器之间的通信是加密的，从而避免了潜在的窃听和中间人攻击。
2. **无错误和警告**：如果你的工具（如 Docker、containerd 或其他与 Harbor 交互的工具）不信任该证书，你通常会看到错误或警告消息，表明证书是不受信任的。配置机器以信任自签名证书后，这些消息将不再出现。
3. **节省成本**：虽然受信任的证书颁发机构（如 Let's Encrypt）提供免费的证书，但在某些内部网络或私有云环境中，与外部服务通信可能是一个挑战。在这种情况下，自签名证书提供了一种简便的方法来加密通信，而不需要与外部实体交互。

接下来我会给安装好的Harbor仓库设置ssl自签名证书，并在安装docker的机器和以containerd作为CRI的k8s集群中设置自签名证书的信任，从而确保拉取Harbor仓库的镜像时不报"x509"相关的错误。
### 1. Harbor集群设置自签名ssl证书
如果不了解如何安装Harbor仓库，请参考：[Kubernetes安装Harbor仓库](https://blog.csdn.net/weixin_46660849/article/details/130934077)

通过上述方式安装的Harbor仓库Nginx Ingress映射出来的域名为"harbor.example.com"，接下来需要为这个域名签发自签名ssl证书，相关签发方式请参考：[Configure HTTPS Access to Harbor](https://goharbor.io/docs/2.0.0/install-config/configure-https/)

我的签发命令为：

```bash
# Generate a Certificate Authority Certificate
openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=GuangDong/L=GuangZhou/O=Yiqi/OU=Personal/CN=harbor.example.com" \
 -key ca.key \
 -out ca.crt
 
# Generate a Server Certificate
openssl genrsa -out domain.key 4096

openssl req -sha512 -new \
 -subj "/C=CN/ST=GuangDong/L=GuangZhou/O=Yiqi/OU=Personal/CN=harbor.example.com" \
 -key domain.key \
 -out domain.csr

cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.example.com
DNS.2=harbor.example
DNS.3=yiqi.centos.com
EOF

openssl x509 -req -sha512 -days 3650 \
 -extfile v3.ext \
 -CA ca.crt -CAkey ca.key -CAcreateserial \
 -in domain.csr \
 -out domain.crt
```
接下来需要将ssl证书配置到ingress中

```bash
# 生成tls类型的secret
kubectl create secret -n har generic myharbor-ingress --from-file=ca.crt=ca.crt --from-file=tls.crt=domain.crt --from-file=tls.key=domain.key

# 查看Harbor ingress配置
kubectl get ing -n har
NAME                    CLASS   HOSTS                ADDRESS        PORTS     AGE
harbor-ingress          nginx   harbor.example.com   172.16.80.22   80, 443   91d
harbor-ingress-notary   nginx   notary.example.com   172.16.80.22   80, 443   91d

# 修改ingress tls使用的secret，在spec.tls.hosts栏位下
  tls:
  - hosts:
    - harbor.example.com
    secretName: myharbor-ingress # 修改为之前生成的tls类型的secret
```
完成后可以通过Chrome浏览器登录查看证书，可以看到虽然Chrome浏览器依然会提示证书不安全，由于不是通过授信的ca机构签发的证书。但是我们自签名证书的信息已经能看到了。到这里就代表"Harbor集群设置自签名ssl证书"已经成功。
![在这里插入图片描述](https://img-blog.csdnimg.cn/3702c5e06222499c96a304ea2a13edea.png)
### 2. 为Linux机器设置证书授信
不同的 Linux 发行版中信任自签名证书的方法可能会有所不同。以下是一些常见 Linux 发行版的指导：

1. **Debian/Ubuntu**:

   - 将你的证书复制到 `/usr/local/share/ca-certificates/`:
     ```bash
     sudo cp your-cert.crt /usr/local/share/ca-certificates/
     ```
   - 更新证书列表:
     ```bash
     sudo update-ca-certificates
     ```

2. **RHEL/Fedora/CentOS**:

   - 将你的证书复制到 `/etc/pki/ca-trust/source/anchors/`:
     ```bash
     sudo cp your-cert.crt /etc/pki/ca-trust/source/anchors/
     ```
   - 更新证书列表:
     ```bash
     sudo update-ca-trust
     ```

3. **openSUSE/SUSE**:

   - 将你的证书复制到 `/etc/pki/trust/anchors/`:
     ```bash
     sudo cp your-cert.crt /etc/pki/trust/anchors/
     ```
   - 更新证书列表:
     ```bash
     sudo update-ca-certificates
     ```

4. **Alpine Linux**:

   - 将你的证书复制到 `/etc/ssl/certs/`:
     ```bash
     sudo cp your-cert.crt /etc/ssl/certs/
     ```
   - 更新证书列表:
     ```bash
     sudo update-ca-certificates
     ```
#### 2.1 Docker所在机器添加证书信任
使用如下命令尝试pull Harbor仓库中公开的镜像，可以看到由于仓库不授信无法拉取镜像。
```bash
docker pull harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685
Error response from daemon: Get "https://harbor.example.com/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority
```
添加ca证书的信任

```bash
# 将之前生成的ca.crt证书拷贝到当前机器
sudo cp ca.crt /etc/pki/ca-trust/source/anchors/

sudo update-ca-trust

sudo systemctl restart docker

# 再尝试拉取镜像
docker pull harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685
harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685: Pulling from library/nginx
f56be85fc22e: Pull complete
2ce963c369bc: Pull complete
59b9d2200e63: Pull complete
3e1e579c95fe: Pull complete
547a97583f72: Pull complete
1f21f983520d: Pull complete
c23b4f8cf279: Pull complete
Digest: sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685
Status: Downloaded newer image for harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685
harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685
```
#### 2.1 containerd所在的K8S几区节点添加证书信任
我之前的文章有介绍过如何为containerd设置非授信仓库如果感兴趣请查看：[containerd 设置非授信仓库](https://blog.csdn.net/weixin_46660849/article/details/132415839)
使用如下命令尝试pull Harbor仓库中公开的镜像，可以看到由于仓库不授信无法拉取镜像。
```bash
crictl pull harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685
E0828 20:41:53.947090   21601 remote_image.go:238] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685\": failed to resolve reference \"harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685\": failed to do request: Head \"https://harbor.example.com/v2/library/nginx/manifests/sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685\": x509: certificate signed by unknown authority" image="harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685"
FATA[0000] pulling image: rpc error: code = Unknown desc = failed to pull and unpack image "harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685": failed to resolve reference "harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685": failed to do request: Head "https://harbor.example.com/v2/library/nginx/manifests/sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685": x509: certificate signed by unknown authority 
```
```bash
# 将之前生成的ca.crt证书拷贝到当前机器
sudo cp ca.crt /etc/pki/ca-trust/source/anchors/

sudo update-ca-trust

sudo systemctl restart containerd

# 再尝试拉取镜像
crictl pull harbor.example.com/library/nginx@sha256:01ccf4035840dd6c25042b2b5f6b09dd265b4ed5aa7b93ccc4714027c0ce5685
Image is up to date for sha256:8e75cbc5b25c8438fcfe2e7c12c98409d5f161cbb668d6c444e02796691ada70
```
