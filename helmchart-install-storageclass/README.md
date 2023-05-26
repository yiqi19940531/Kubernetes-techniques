## Kubernetes集群通过NFS Subdirectory External Provisioner安装StorageClass
### 基本介绍
k8s集群1.26.4在Centos7.9机器上基于Kubeadm安装。通过NFS Subdirectory External Provisioner(Helm Chart方式)安装StorageClass，本文将介绍相关步骤及注意事项。NFS Subdirectory External Provisioner官方说明此种方式适合Kubernetes >=1.9
### 环境介绍
机器名     | ip | 系统版本 | 系统内核 | cpu架构
-------- | -----  | -----   | -------  | -------
m1  | 10.11.81.152  | CentOS Linux release 7.9 |   5.4.242-1.el7 |   x86
w1  | 10.11.81.153  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86
w2  | 10.11.81.154  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86
w3  | 10.11.81.155  | CentOS Linux release 7.9 |    5.4.242-1.el7 |   x86
nfs  | 10.11.82.6  | CentOS Linux release 7.9 |    3.10.0-1160.90.1 |   x86
### 1.安装NFS服务端（nfs机器上操作）
1.1安装 NFS 服务器所需的软件包
```bash
yum install -y nfs-utils
```
1.2设置 NFS 服务开机启动(必须先启动rpcbind服务)
```bash
systemctl enable rpcbind
systemctl enable nfs
```
1.3启动nfs服务
```bash
systemctl start rpcbind
systemctl start nfs
```
1.4如果有防火墙服务，需要打开 rpc-bind 和 nfs 的服务
```bash
firewall-cmd --zone=public --permanent --add-service={rpc-bind,mountd,nfs}
firewall-cmd --reload
```
### 2.配置共享目录（nfs机器上操作）
2.1 NFS服务启动之后，在服务端配置一个共享目录
```bash
mkdir /nfs-share
```
2.2配置 NFS 服务器共享的目录
```bash
echo "/nfs-share    10.11.81.0/24(rw,sync,no_root_squash,no_all_squash) > /etc/exports
```
a./nfs-share: 共享目录位置
b.10.11.81.0/24: 客户端 IP 范围，* 代表所有，即没有限制。如果是生产环境建议严格控制可访问IP
c.rw: 权限设置，可读可写
d.sync: 同步共享目录
e.no_root_squash: 可以使用 root 授权
f.no_all_squash: 可以使用普通用户授权
2.3使NFS配置生效
```bash
exportfs -r
```
2.4查看挂载情况
```bash
exportfs
```
### 3.安装Helm Chart（m1机器上操作）
[官方网站](https://helm.sh/docs/intro/install/)
3.1如果不参考官网，直接执行以下命令安装Helm Chart
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

#测试是否安装成功
helm version
```
### 4.所有工作节点安装nfs-utils（w*节点上操作）
遇到错误：[参考文章](https://aijishu.com/a/1060000000096159)
如果不执行这一步在第5步“安装nfs-provisioner”启动nfs-安装nfs-provisioner容器时会出现如下错误：
```bash
mount: wrong fs type, bad option, bad superblock on 125.64.41.244:/data/img,
       missing codepage or helper program, or other error
       (for several filesystems (e.g. nfs, cifs) you might
       need a /sbin/mount.<type> helper program)
       In some cases useful info is found in syslog - try
       dmesg | tail  or so
```
在网上搜索了一下说是没有安装 mount.nfs导致，也是说需要在nfs工作节点安装mount.nfs就不会有wrong fs type, bad option, bad superblock错误提示了。

根据错误提示，查看/sbin/mount.<type>文件，果然发现没有/sbin/mount.nfs的文件，安装nfs-utils即可。
```bash
#所有工作节点执行
yum install nfs-utils -y
```
### 5.安装nfs-provisioner（m1机器上操作）
[官网参考](https://artifacthub.io/packages/helm/nfs-subdir-external-provisioner/nfs-subdir-external-provisioner)
```bash
#添加repo
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=10.11.82.6 \   #nfs服务提供方机器
    --set nfs.path=/nfs-share \     #nfs服务提供方目录
    --set storageClass.name=nfs-client \  #Storage Class名称
    --set storageClass.defaultClass=true  #设置为默认Storage Class
```
注意：默认安装的nfs-subdir-external-provisioner镜像来源于	registry.k8s.io，国内下载可能受限，可以通过如下地址下载相关镜像：
https://download.csdn.net/download/weixin_46660849/87822652
```bash
#配置时可以指定镜像仓库
--set image.repository=<可以下载nfs-subdir-external-provisioner的本地或远程镜像仓库>
```
### 6.测试验证（m1机器上操作）
6.1确保nfs-provisioner容器正确启动
```bash
#添加repo
kubectl get po
#看到如下输出
NAME                                              READY   STATUS    RESTARTS   AGE
nfs-subdir-external-provisioner-456ed7c90-acjn1   1/1     Running   0          2m
```
6.2验证pvc挂在
```bash
cat <<EOF >  pvc-test.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  storageClassName: "nfs-client"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
EOF
```
