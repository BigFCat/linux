# 容器技术概述及安装
## 什么是容器技术？
以前学习KVM，同时也了解了些docker，懵懂状态，没区分两者的优缺点。第一家公司有十几台 kvm宿主机，配置都挺好的，主要用来跑java，管理很麻烦，并且接手的时候也没有看到监控相关的配置！这也算是我最开始接触虚拟化技术吧，干了15天就离职了，因为盘子太大了，前人离职留的文档太少了，项目也不景气，很无赖。不管什么好技术，使用的人没规划好没正规化，都是坑吧。

现在开始使用docker，同时使用k8s编排，给我的感觉就是像手机安装app一样，一个资源干一件事，分的很清楚，不管是权限，还是资源控制；之前使用的kvm，做一个不太恰当的比喻，就像是你使用微信是一台手机，使用微博又需要一台手机。

容器是共享底层操作系统(内核空间)，启动无需启动整个操作系统。

Docker 使用 Google 公司推出的 Go 语言 进行开发实现，基于 Linux 内核的 cgroup，namespace，以及 AUFS 类的 Union FS 等技术，对进程进行封装隔离，属于 操作系统层面的虚拟化技术。由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。最初实现是基于 LXC，从 0.7 版本以后开始去除 LXC，转而使用自行开发的 libcontainer，从 1.11 开始，则进一步演进为使用 runC 和 containerd。

Docker 在容器的基础上，进行了进一步的封装，从文件系统、网络互联到进程隔离等等，极大的简化了容器的创建和维护。使得 Docker 技术比虚拟机技术更为轻便、快捷。

## 为什么要使用docker
1. **轻量化** 容器首先是轻量化，不像kvm，一台虚拟机就是一个系统。轻量化我的理解是镜像的轻量，[dockerhub](https://hub.docker.com/)官方提供的各种镜像可以去看一看，我用[node的官方镜像](https://hub.docker.com/_/node)  举例:![20190417103436](https://img.24linux.com/static/images/20190417103436.png) 图中10.15.3的版本分别有基于`alpine`,`jessie`,`jessie-slin`,`stretch`为基础镜像构建的node镜像。从大小上来讲，alpine最小、slim稍大、默认的最大。~~所以应该尽可能的使用alpine版本的，如果发现程序的运行环境缺少某些东西，那么尝试用slim版本或者默认版本~~。
2. **移植性** docker容器在构建后就将程序及依赖的运行环境都打包成一个镜像，可以运行于支持docker容器引擎的所有操作系统上。这使得运维工程师只需要搭建安装docker服务，项目完成后只要dockerfile编写好，就可以直接运行。能运行的docker平台目前可以是公有云，私有云，物理机，虚拟机，还可以是你的笔记本。
3. **一致的运行环境** “我在我电脑上运行的时候没问题啊”，这种情况基本可以告别了，因为运行环境都打包在镜像中了，在测试服测试完后，你可能只需要改变切换分支到生产分支上，重新打包镜像就能部署到生产服务器中。
4. **持续部署/交付** 使用编排工具，如k8s，swarm等，结合jenkins，drone等工具使用 Docker 可以通过定制应用镜像来实现持续集成、持续交付、部署。
5. **更快的启动时间** kvm分钟级别，容器秒级别。

## Centos安装 Docker-CE
1. 安装必要的一些系统工具
```
yum install -y yum-utils device-mapper-persistent-data lvm2
```
2. 添加软件源信息
```
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 默认未开启 test版本和 nightly版本，如果需要研究学习，可以参考官网安装 https://docs.docker.com/install/linux/docker-ce/centos/
```
3. 更新并安装 Docker-CE
```
yum makecache fast 
yum -y install docker-ce
```
4. 开启Docker服务
```
service docker start
```

## 安装指定版本的 Docker-CE:
1. 查找Docker-CE的版本:
```
yum list docker-ce.x86_64 --showduplicates | sort -r
```
```
Loading mirror speeds from cached hostfile
Loaded plugins: branch, fastestmirror, langpacks 
docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable 
docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable 
Available Packages
```
2. 安装指定版本的Docker-CE: \(VERSION 例如上面的17.03.0.ce.1-1.el7.centos\)
```
yum -y install docker-ce-[VERSION]
```
3. 安装校验：
```
docker version
```

## 配置加速器：
加速器推荐阿里的，申请链接 https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
```
#systemctl enable docker（产生下面的配置文件，为什么？可以去阮一峰老师博客看看systemd得教程）

#vim /etc/systemd/system/multi-user.target.wants/docker.service
将ExecStart=/usr/bin/dockerd
修改为
ExecStart=/usr/bin/dockerd --registry-mirror=https://registry.docker-cn.com
此为阿里云加速器，执行 ps -ef | grep docker 或者docker info 也可以看见加速ok
```

也可以这样修改：(适用于systemd的系统)

```
#vim /etc/docker/daemon.json 
{
            "registry-mirrors": ["https://registry.docker-cn.com"],
            "insecure-registries": ["x.x.x.x:8083","x.x.x.x:8082"],   # 在使用私有仓库时，需要添加的私有仓库地址，原因是因为没有ssl认证
            "disable-legacy-registry": true
}
```

```
#systemctl daemon-reload
#systemctl start docker
```

## 安装docker-compose
1. 运行此命令以下载Docker Compose的当前稳定版本：
```
curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
2. 对二进制文件应用可执行权限
```
chmod +x /usr/local/bin/docker-compose
```