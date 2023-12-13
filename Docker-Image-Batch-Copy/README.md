## docker save多个镜像打包成一个tar.gz压缩文件
有时候我们需要将docker中的多个镜像批量的传输到另一台机器，如果通过docker save这种命令则需要制作多个tar文件，这样以来冗余的操作较多而且tar文件占据的空间较大，不利于传输。

可以通过以下命令在两个docker之间实现多个镜像批量传输：
```bash
# 原机器
docker save image1:tag1 image2:tag2 <可以加入更多> | gzip > images.tar.gz

# 目标机器
gunzip -c images.tar.gz | docker load
```

如果想将所有镜像传输到另一台机器可以使用以下命令：
```bash
# 原机器
images=$(docker images --format '{{.Repository}}:{{.Tag}}')
docker save $images | gzip > images.tar.gz

# 目标机器
gunzip -c images.tar.gz | docker load
```
