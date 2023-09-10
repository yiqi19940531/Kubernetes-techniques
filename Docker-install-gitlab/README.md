## 基于Docker安装GitLab

GitLab 是一个为软件开发设计的开源工具，提供了 Git 版本控制、持续集成、DevOps、和软件部署等功能。GitLab 可以被认为是一个完整的 DevOps 平台，通过一个单一的应用来实现。
以下是 GitLab 的一些核心功能：

1.版本控制 (Git Repository Management)
提供了一个基于 Web 的 Git 仓库管理界面。
支持代码评审、评论、分支管理和合并请求。

2.持续集成与持续部署 (CI/CD)
允许开发者自动测试和部署代码，以确保代码的质量和部署的速度。
提供了一个内建的容器仓库。

3.自动 DevOps
提供了自动的配置选项，使得从代码到生产的整个流程更加自动化。
   
4.Kubernetes 集成
支持 Kubernetes，使得部署、扩展和管理容器应用更加简便。

5.代码安全与合规性
提供了静态和动态的安全测试。
提供了依赖性扫描，确保项目的依赖是安全的。
代码质量分析和代码漂移分析。

6.问题跟踪 & 事务管理
提供了一个内建的问题跟踪器。
支持看板、里程碑、时间跟踪和优先级标签。

7.用户和访问管理
支持多用户、多角色和访问级别，确保合适的团队成员能够访问合适的资源。

GitLab 不仅提供了一个社区版（CE，Community Edition），而且还有一些付费版本，其中包括了更多高级和企业级功能。

如果是规范化的企业级应用，我推荐在Kubernetes中安装，这个我后期也会写文章介绍通过Helm Chart来安装。本文主要介绍Docker中的安装，这种方式非常简单。
## 安装GitLab流程
设置卷位置
在设置其他所有内容之前，请配置一个新的环境变量 $GITLAB_HOME，指向配置、日志和数据文件所在的目录。 确保该目录存在并且已授予适当的权限（具有管理员的rwx权限）。

```bash
# 创建目录，赋予权限
mkdir -p /root/gitlab
export GITLAB_HOME=/root/gitlab

# 定义启动时环境变量
echo "export GITLAB_HOME=/root/gitlab" >> ~/.bash_profile
source ~/.bash_profile
```
Docker启动命令
```bash
# 其中端口号要结合实际情况，但是对于Docker中安装很多时候远程服务器的22端口早就被占中，所以这里宿主机选择的是2289端口
sudo docker run --detach \
  --hostname gitlab.yiqi.com \
  --publish 443:443 --publish 80:80 --publish 2289:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/opt/gitlab \
  --shm-size 256m \
  registry.gitlab.cn/omnibus/gitlab-jh:latest

# gitlab单体镜像比较大，启动过程比较慢，可以查看STATUS中的状态(health: xxx)
docker ps
CONTAINER ID   IMAGE                                                              COMMAND                  CREATED             STATUS                            PORTS                                                            NAMES
njnsdjvgyvbd   registry.gitlab.cn/omnibus/gitlab-jh:latest                        "/assets/wrapper"        10 seconds ago      Up 7 seconds (health: starting)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:2289->22/tcp   gitlab
```
页面登录

```bash
# 经过一段时间等待后，GitLab容器启动成功
docker ps
CONTAINER ID   IMAGE                                                              COMMAND                  CREATED             STATUS                    PORTS                                                            NAMES
njnsdjvgyvbd   registry.gitlab.cn/omnibus/gitlab-jh:latest                        "/assets/wrapper"        12 minutes ago      Up 12 minutes (healthy)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:2289->22/tcp   gitlab

# 查看初始密码“docker exec -it 容器名字或容器ID grep 'Password:' /etc/gitlab/initial_root_password”
docker exec -it njnsdjvgyvbd grep 'Password:' /etc/gitlab/initial_root_password
Password: 4VksSgrbaflg2cY5K/qgfulb2CkbKfR/KuiYZoBx2pE=
```
页面登录
![在这里插入图片描述](https://img-blog.csdnimg.cn/f2eff33251ee48f998b902633cb03f0e.png)
登录后改密码
![在这里插入图片描述](https://img-blog.csdnimg.cn/142d60fe72a249f2897b8b93fe881790.png)
