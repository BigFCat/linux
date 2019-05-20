# docker基本概念-镜像image-容器container-仓库registry

还是用手机上的app应用做类比，image相当于软件商城的软件，你需要下载；安装运行后就是container；而软件商城就是仓库registry

## 镜像

操作系统分 `内核`和 `用户空间` 。对于 Linux 而言，内核启动后，会挂载 root 文件系统为其提供用户空间支持。而 Docker 镜像（Image），就相当于是一个 root 文件系统。上一篇文章中提到的node-alpine镜像也是一个精简的文件系统。

Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

在[docker hub](https://hub.docker.com)中有很多官方镜像，也可以基于alpine等基础镜像自主构建。

## 容器

将镜像run起来就会生成一个容器，容器可以创建，销毁，暂停。但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间。因此容器可以拥有自己的 root 文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。这种特性使得容器封装的应用比直接在宿主运行更加安全。

容器实现进程隔离和资源隔离的原理，可以看下面文章简单了解。

[DOCKER基础技术：LINUX CGROUP](https://coolshell.cn/articles/17049.html)

[DOCKER基础技术：LINUX NAMESPACE（上）](https://coolshell.cn/articles/17010.html)

[DOCKER基础技术：LINUX NAMESPACE（下）](https://coolshell.cn/articles/17029.html)

## 仓库

用于存储镜像，常见的有Docker Registry，habor，nexus3。阿里云现在也提供镜像仓库，使用起来也比较方便，但是如果用于ci/cd 还是得搭建内网私有仓库，毕竟有时候一个镜像也得100M+