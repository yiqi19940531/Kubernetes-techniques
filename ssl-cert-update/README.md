## 基于Kubeadm安装的Kubernetes集群证书更新
### Kubernetes集群中常用的证书介绍
在通过`kubeadm`安装的Kubernetes集群中，所有集群相关的证书和私钥默认都存放在Kubernetes控制平面节点的`/etc/kubernetes/pki`目录下。对于那些由`kubeadm`创建的证书，默认的有效期是1年，之后需要更新。以下是一些主要的证书及其用途：

1. **CA (Certificate Authority) 证书和私钥** (`ca.crt` 和 `ca.key`)：这是一个自签名的根证书，用来签署集群中其他证书。所有的其他证书都会通过这个CA来建立信任链。

2. **APIServer 证书** (`apiserver.crt` 和 `apiserver.key`)：用于Kubernetes API服务器的TLS认证。当任何客户端（如`kubectl`、集群内服务等）尝试与Kubernetes API服务器通信时，都会使用这个证书来加密通信。

3. **APIServer kubelet 客户端证书** (`apiserver-kubelet-client.crt` 和 `apiserver-kubelet-client.key`)：API服务器用来与集群中每个kubelet进行安全通信的客户端证书。

4. **APIServer edcd 客户端证书** (`apiserver-etcd-client.crt` 和 `apiserver-etcd-client.key`)：API服务器用来与etcd通信的客户端证书和私钥，这个证书用于API服务器验证它对etcd的访问是经过授权的，确保通信是加密和安全的。

5. **前端代理证书** (`front-proxy-ca.crt` 和 `front-proxy-ca.key`)：用于签署前端代理的客户端和服务器证书，前端代理是一个API聚合层。

6. **前端代理客户端证书** (`front-proxy-client.crt` 和 `front-proxy-client.key`)：API服务器用来与前端代理通信的客户端证书。

7. **etcd 证书** (`etcd/server.crt` 和 `etcd/server.key`)：用于`etcd`成员之间以及API服务器与`etcd`通信的TLS认证。

8. **etcd CA 证书** (`etcd/ca.crt` 和 `etcd/ca.key`)：用来签署etcd相关的证书，如etcd成员证书和客户端证书。

9. **etcd 对等证书** (`etcd/peer.crt` 和 `etcd/peer.key`)：用于etcd实例之间的相互TLS认证。

10. **etcd 客户端证书** (`etcd/healthcheck-client.crt` 和 `etcd/healthcheck-client.key`)：API服务器用来进行etcd存活性检查的客户端证书。

11. **Controller Manager 客户端证书** (`controller-manager.conf` 内嵌了 `controller-manager.crt` 和 `controller-manager.key`)：Kubernetes控制器管理器用来与API服务器通信的客户端证书。

12. **Scheduler 客户端证书** (`scheduler.conf` 内嵌了 `scheduler.crt` 和 `scheduler.key`)：Kubernetes调度器用来与API服务器通信的客户端证书。

13. **admin 用户证书** (`admin.conf` 内嵌了 `admin.crt` 和 `admin.key`)：允许管理员通过`kubectl`与API服务器通信的客户端证书。

14. **kubelet.conf 用户证书** (`kubelet.conf` 其中包含 `kubelet.crt` 和 `kubelet.key`)：相关证书在“/var/lib/kubelet/pki/”路径下，这是每个节点上kubelet的配置文件，用于kubelet身份认证的客户端证书和私钥。这些证书允许kubelet与API服务器安全地通信。

如果想对Kubernetes集群中证书的作用有更加深刻的理解推荐您阅读 [一文带你彻底厘清Kubernetes中的证书工作机制](zhaohuabing.com/post/2020-05-19-k8s-certificate/)

### Kubernetes集群重新签发10年有效期证书
详细的操作步骤请参考这个GitHub项目  [Kubernetes证书更新](https://github.com/yuyicai/update-kube-cert/blob/master/README-zh_CN.md)，我在多个版本测试有效。
