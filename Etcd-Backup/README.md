## Kubernetes集群etcd备份
Kubernetes 集群中，etcd 存储了所有集群的状态和配置数据，包括 Pod、服务、和部署的信息。因此，定期备份 etcd 是维护 Kubernetes 集群稳定性和可靠性的重要部分。
### etcd常见的运行方式
在 Kubernetes 集群中，etcd 通常被用作保存所有集群数据的关键值存储系统。etcd 的安装和配置方式在不同的 Kubernetes 集群部署方案中有所差异，但主要可以分为以下几种：
###  **集成在 Kubernetes 控制平面节点上**
在许多 Kubernetes 安装工具（如 kubeadm）和服务（例如一些托管的 Kubernetes 服务）中，etcd 通常作为集群的一部分直接运行在控制平面节点上。在这种情况下，etcd 可以：

- 作为静态 Pod 运行，由 kubelet 直接管理。
- 作为系统服务运行（例如，使用 systemd）。
- 在一些托管的 Kubernetes 服务中，etcd 的管理和维护由服务提供商负责，对用户来说是透明的。

备份etcd的根本还是要能访问etcd对外服务的api地址和端口，以及有操作etcd的工具etcdctl
### 1.确认当前集群是否安装了etcdctl工具
**在任意Master节点**
```bash
# 命令行执行'etcdctl version'命令如果出现以下提示则代表当前机器已经安装etcdctl工具
$ etcdctl version
etcdctl version: 3.5.10
API version: 3.5
```

> 通常如果etcd作为系统服务则通常会安装好etcdctl工具，如果etcd是以容器的方式运行则不会安装
### 2.安装etcdctl工具
如果未安装etcdctl工具则需要进行安装。
#### 2.1 下载 etcd 二进制包
首先，访问 [etcd 的 GitHub 发布页面](https://github.com/etcd-io/etcd/releases) 来找到最新的稳定版本。你可以使用 `wget` 或 `curl` 命令下载。例如：

```sh
ETCD_VER=v3.5.10  # 替换为你需要的版本

# 下载 etcd
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz

# 解压文件
tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
```

#### 2.2 安装etcdctl
解压后，你会得到一个包含 `etcd` 和 `etcdctl` 的目录。

```sh
cd etcd-${ETCD_VER}-linux-amd64

# 将 etcdctl 移动到一个全局路径，例如 /usr/local/bin
sudo mv etcdctl /usr/local/bin/
```
#### 2.3验证安装
安装完成后，你可以通过运行以下命令来检查 `etcdctl` 的版本，以确认正确安装：

```sh
etcdctl version
```
### 3.确认本机有etcd对外提供服务的端口
etcd对外提供服务的端口通常是2379
```bash
# 通过执行“netstat -antp | grep 2379”看到"127.0.0.1:2379"则表示本机的2379端口对外提供服务
$ netstat -antp | grep 2379
tcp        0      0 9.46.255.229:2379       0.0.0.0:*               LISTEN      1545/etcd           
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      1545/etcd 
```
### 4.确认etcd证书路径
备份etcd数据时会用到以下证书
- **etcd 服务器证书**
- **etcd 服务器密钥**
- **etcd CA（证书颁发机构）证书**

绝大多数情况这些证书会存放在"/etc/kubernetes/pki/etcd/"路径下。

> 某些作为系统服务运行的etcd证书可能不在常规目录下，可以通过“ps aux | grep etcd”命令查看相关证书的地址。
### 5.备份etcd数据

```bash
cd ~ && mkdir etcd-backup
cd etcd-backup

# 假设证书在默认路径“/etc/kubernetes/pki/etcd/”
etcdctl --endpoints 127.0.0.1:2379  \
--cert="/etc/kubernetes/pki/etcd/server.crt"  \
--key="/etc/kubernetes/pki/etcd/server.key"  \
--cacert="/etc/kubernetes/pki/etcd/ca.crt"   \
snapshot save ./snap-$(date +%Y-%m-%d).db
```
### 6.查看快照状态
```bash
etcdctl snapshot status snap-2023-10-29.db --write-out=table
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 5759c486 |    25395 |        952 |     3.7 MB |
+----------+----------+------------+------------+ 
-----------------------------------
```
