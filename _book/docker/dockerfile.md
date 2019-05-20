# dockerfile 编写

[官方dockerfile编写参考](https://docs.docker.com/engine/reference/builder/)

在最开始学习docker的时候，没区分docker和kvm的区别，往往是容器跑起来后exec容器，然后使用yum或者其他方式搭建环境，现在想想特别傻。但是这种问题总会遇到，基础镜像中并不能满足你的需求，就要自己构建镜像。

构建镜像如果采用最开始学习的傻方法不仅不是正确的使用方法，也会造成一个问题--镜像不够透明。时间久了。你也不知道最初进行了怎样的构建！

Dockerfile就是解决这问题，Dockerfile本身是一个文本文档，同时也可以将繁琐的搭建转化为代码形式，维护也会更加方便。

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

## 使用Dockerfile的原理

在安装好docker后(安装参考链接)，使用`docker version`命令可以看到显示了server和client两个服务的版本信息

```shell
# docker version
Client:
 Version:      18.03.1-ce
 API version:  1.37
 Go version:   go1.9.5
 Git commit:   9ee9f40
 Built:        Thu Apr 26 07:20:16 2018
 OS/Arch:      linux/amd64
 Experimental: false
 Orchestrator: swarm

Server:
 Engine:
  Version:      18.03.1-ce
  API version:  1.37 (minimum version 1.12)
  Go version:   go1.9.5
  Git commit:   9ee9f40
  Built:        Thu Apr 26 07:23:58 2018
  OS/Arch:      linux/amd64
  Experimental: false

```
Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API，被称为 Docker Remote API，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。也因为这种 C/S 设计，让我们操作远程服务器的 Docker 引擎变得轻而易举。

当我们进行镜像构建的时候，并非所有定制都会通过 RUN 指令完成，经常会需要将一些本地文件复制进镜像，比如通过 COPY 指令、ADD 指令等。而 docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？

这就引入了上下文的概念。当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

## 以nginx官方Dockerfile构建镜像

我使用的是 官方nginx:1.15.12的Dockerfile https://raw.githubusercontent.com/nginxinc/docker-nginx/e5123eea0d29c8d13df17d782f15679458ff899e/mainline/alpine/Dockerfile

使用 `docker build`构建镜像

docker build -t 镜像名称 .     可以指定本地文件

docker build -t 镜像名称 -f dockerfile .    可以指定dockerfile在文件系统中的位置

docker build dockerfile-http   可以指定网络文件

```shell
# docker build https://raw.githubusercontent.com/nginxinc/docker-nginx/e5123eea0d29c8d13df17d782f15679458ff899e/mainline/alpine/Dockerfile
Sending build context to Docker daemon  7.168kBs://raw.githubusercontent.com/nginxinc/docker-nginx/e5123eea0d29c8d13df17d782f15679458ff899e/mainline/alpine/Dockerfile  5.402kB
Step 1/9 : FROM alpine:3.9
 ---> cdf98d1859c1
Step 2/9 : LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"
 ---> Running in 5963725fb3bf
Removing intermediate container 5963725fb3bf
 ---> c99cf680834c
Step 3/9 : ENV NGINX_VERSION 1.15.12
 ---> Running in e1e9c712280d
...
...
Executing busybox-1.29.3-r10.trigger
OK: 15 MiB in 29 packages
Removing intermediate container ae5177199681
 ---> 86088840b395
Step 5/9 : COPY nginx.conf /etc/nginx/nginx.conf
COPY failed: stat /var/lib/docker/tmp/docker-builder548521117/nginx.conf: no such file or directory
```

```shell
# docker build -t nginx -f ./Dockerfile.nginx  .
Sending build context to Docker daemon  8.192kB
Step 1/9 : FROM alpine:3.9
 ---> cdf98d1859c1
Step 2/9 : LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"
 ---> Running in 04ffef3125b9
Removing intermediate container 04ffef3125b9
 ---> ce7374c0d7a3
Step 3/9 : ENV NGINX_VERSION 1.15.12

```
## 上下文 context

以上是两种不同的build方式，一种是采用云上的dockerfile文件，一个是采用本地的文件

注意第二种方式，命令行最后的 “.”  这个是指定`上下文路径` ，不是指定dockerfile文件的路径，指定dockerfile文件路径使用`-f`参数，查看帮助也有简单的说明

```
# docker build -h

Usage:	docker build [OPTIONS] PATH | URL | -

Options:

  -f, --file string             Name of the Dockerfile (Default is 'PATH/Dockerfile')

```

上下文context的作用我理解为告诉docker server的工作目录是哪个，这样就可以将dockerfile种需要ADD，COPY的文件放在工作目录种。为什么会很自然的将 `.` 错误理解为指定dockerfile的位置，因为在不指定dockerfile文件位置时，使用`docker build .` 默认就是使用上下文目录下默认的`Dockerfile`文件

下面使用示例来理解上下文的用处：

我在Dockerfile中 COPY文件到镜像中，这个文件在宿主机的绝对地址为`/root/docker/nginx/nginx.conf`

```shell
[root@VM_79_16_centos nginx]# pwd
/root/docker/nginx
[root@VM_79_16_centos nginx]# cat Dockerfile 
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf

[root@VM_79_16_centos nginx]# docker build .
Sending build context to Docker daemon  3.072kB
Step 1/2 : FROM nginx:alpine
 ---> 0be75340bd9b
Step 2/2 : COPY nginx.conf /etc/nginx/conf
 ---> Using cache
 ---> b1276e473d48
Successfully built b1276e473d48  

### COPY成功，构建镜像成功，文件地址为以上下文目录为根路径

[root@VM_79_16_centos nginx]# vim Dockerfile 
[root@VM_79_16_centos nginx]# cat Dockerfile 
FROM nginx:alpine
COPY /root/docker/nginx/nginx.conf /etc/nginx/conf
### 修改为以系统根路径的绝对路径
[root@VM_79_16_centos nginx]# docker build .
Sending build context to Docker daemon  3.072kB
Step 1/2 : FROM nginx:alpine
 ---> 0be75340bd9b
Step 2/2 : COPY /root/docker/nginx/nginx.conf /etc/nginx/conf
COPY failed: stat /var/lib/docker/tmp/docker-builder587730389/root/docker/nginx/nginx.conf: no such file or directory
### 构建失败，提示文件未找到 ！ 
```

以上示例就能说明上下文的作用。

## .dockerignore 文件

在docker CLI将上下文发送到docker守护程序之前，它会查找.dockerignore在上下文的根目录中指定的文件。如果此文件存在，CLI将修改上下文以排除与其中的模式匹配的文件和目录。

现在我将一个目录COPY到镜像中，使用.dockerignore排除其中的xxx.conf文件

```shell
# cat .dockerignore 
xxxx/xxx.conf
```

```shell
[root@VM_79_16_centos nginx]# ll xxxx/
total 8
-rw-r--r-- 1 root root 4 Apr 18 11:53 nginx.conf
-rw-r--r-- 1 root root 4 Apr 18 12:10 xxx.conf
[root@VM_79_16_centos nginx]# cat Dockerfile 
FROM nginx:alpine
COPY xxxx /etc/nginx/conf

[root@VM_79_16_centos nginx]# docker build .
Sending build context to Docker daemon  4.608kB
Step 1/2 : FROM nginx:alpine
alpine: Pulling from library/nginx
bdf0201b3a05: Pull complete 
08d74e155349: Pull complete 
a9e2a0b35060: Pull complete 
d9e2304ab8d0: Pull complete 
Digest: sha256:61e3db30b1334b1fa0a2e71b86625188f76653565d515d5f74ecc55a8fb91ce9
Status: Downloaded newer image for nginx:alpine
 ---> 0be75340bd9b
Step 2/2 : COPY xxxx /etc/nginx/conf
 ---> 829052bdb86a
Successfully built 829052bdb86a
### 构建镜像  将xxxx目录COPY到镜像目录中

[root@VM_79_16_centos nginx]# docker run -d 829
073790d1e2d340ca792d60e2239dc0f691248be36dfb5dad458568b03dd9aaf6
[root@VM_79_16_centos nginx]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
073790d1e2d3        829                 "nginx -g 'daemon of…"   3 seconds ago       Up 2 seconds        80/tcp              pensive_stallman
[root@VM_79_16_centos nginx]# docker exec -it 073 sh
/ # ls etc/nginx/conf/
nginx.conf

### 进入容器未看到xxx.conf文件
```


## FROM  指定基础镜像

在 Docker Hub 上有非常多的高质量的官方镜像，有可以直接拿来使用的服务类的镜像，如 nginx、redis、mongo、mysql、httpd、php、tomcat 等；也有一些方便开发、构建、运行各种语言应用的镜像，如 node、openjdk、python、ruby、golang 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。

如果没有找到对应服务的镜像，官方镜像中还提供了一些更为基础的操作系统镜像，如 ubuntu、debian、centos、fedora、alpine 等，这些操作系统的软件库为我们提供了更广阔的扩展空间。

除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 `scratch` 。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。

```yaml
FROM scratch
...
```
如果你以 `scratch` 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。

基础操作系统镜像推荐 `alpine` 5M不到。


## LABEL  标签

```yaml
LABEL multi.label1="value1" \
      multi.label2="value2" \
      other="value3"
```
推荐使用以上写法，很情绪。当然最好只写一个LABEL，镜像层数最小化。镜像打标签的作用也是用于管理镜像。


## RUN  执行命令

```yaml
RUN buildDeps='gcc libc6-dev make wget' \ 
    apt-get update && apt-get install -y \
    $buildDeps \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
 && rm -rf /var/lib/apt/lists/* \
 && apt-get purge -y --auto-remove $buildDeps
```

反斜杠分隔拆分长或复杂语句，以使Dockerfile更具可读性，可理解性和可维护性；每一个指令就会增加镜像层数，`Union FS` 是有最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是不得超过 127 层。

`buildDeps` 变量指定了镜像构建的依赖，在最后使用 `apt-get purge` 命令移除依赖和产生的配置文件。`apt-get remove` 只会移除安装的依赖，配置文件并不会删除，使用`purge` 可以使镜像更加精简。


## COPY 复制文件

格式：

```
COPY [--chown=<user>:<group>] <源路径>... <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]
```

和 RUN 指令一样，也有两种格式，一种类似于命令行，一种类似于函数调用。

<源路径> 可以是多个，甚至可以是通配符，其通配符规则要满足 Go 的 [filepath.Match](https://golang.org/pkg/path/filepath/#Match) 规则，如：

```yaml
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

<目标路径> 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 `WORKDIR` 指令来指定）。目标路径不需要事先创建，如果目录不存在会在复制文件前先行创建缺失目录。

此外，还需要注意一点，使用 COPY 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。这个特性对于镜像定制很有用。特别是构建相关文件都在使用 Git 进行管理的时候。

在使用该指令的时候还可以加上 --chown=<user>:<group> 选项来改变文件的所属用户及所属组。

```yaml
COPY --chown=55:mygroup files* /mydir/
COPY --chown=bin files* /mydir/
COPY --chown=1 files* /mydir/
COPY --chown=10:11 files* /mydir/
```

复制包含特殊字符（例如[ 和 ]）的文件或目录时，需要按照Golang规则转义这些路径，以防止它们被视为匹配模式。例如，要复制名为的文件arr[0].txt，请使用以下命令;

```
COPY arr[[]0].txt /mydir/    # copy a file named "arr[0].txt" to /mydir/
```
这里不是很明白是怎么转义的，来自[官方文档](https://docs.docker.com/engine/reference/builder/#copy)

注意： 

COPY 和 ADD 命令具有相同的特点：只复制目录中的内容而不包含目录自身，如果需要复制目录到镜像中，需要在目标路径中指定这个目录的名称。

## ADD 

在 Docker 官方的 [Dockerfile 最佳实践文档](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) 中要求，尽可能的使用 COPY，因为 COPY 的语义很明确，就是复制文件而已，而 ADD 则包含了更复杂的功能，其行为也不一定很清晰。最适合使用 ADD 的场合，就是需要自动解压缩的场合。

## WORKDIR   指定工作目录

WORKDIR 命令为后续的 RUN、CMD、COPY、ADD 等命令配置工作目录。在设置了 WORKDIR 命令后，接下来的 COPY 和 ADD 命令中的相对路径就是相对于 WORKDIR 指定的路径。即使dockerfile后面的指令没使用 WORKDIR，工作目录也将创建。

```yaml
# cat Dockerfile 
FROM nginx:alpine
WORKDIR /app
COPY xxxx .
COPY 1[[]0].txt /etc/nginx/conf
### Dockerfile文件  创建工作目录 将xxxx目录中的文件复制到工作目录下

# docker exec -it fe sh
/app # ls
nginx.conf
/app # exit
### 进入容器后直接进入app工作目录
```

该 WORKDIR 指令可以在a中多次使用Dockerfile。如果提供了相对路径，则它将相对于前一条WORKDIR指令的路径 。例如：

```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```
最终pwd命令的输出Dockerfile将是 `/a/b/c`

该 WORKDIR 指令可以解析先前使用的环境变量 ENV。例如：

```
ENV DIRPATH /path
WORKDIR $DIRPATH/$DIRNAME
RUN pwd
```

最终pwd命令的输出Dockerfile将是 `/path/$DIRNAME`


## ENV 设置容器内环境变量

格式有两种：

```yaml
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```

这个指令很简单，就是设置环境变量而已，无论是后面的其它指令，如 RUN，还是运行时的应用，都可以直接使用这里定义的环境变量。

```yaml
ENV VERSION=1.0 DEBUG=on \
    NAME="Hello world"
```

这个例子中演示了如何换行，以及对含有空格的值用双引号括起来的办法，这和 Shell 下的行为是一致的.


## ARG 设置构建镜像时的环境变量

**ARG 和 ENV 的关系**

构建参数和 ENV 的效果一样，都是设置环境变量。不同的是，ARG 所设置的构建环境的环境变量.

还要注意的是，在Dockerfile中，想要在 FROM 引用 ARG 的值，ARG参数必须写在 FROM之前 ，并且这个写在 FROM 之前的ARG值此时只能在FROM中引用，如果后面的构建过程需要引用，必须再重新申明。如下

下面示例是在FROM前后都使用 `ARG 指令`

```yaml
ARG VERSION
FROM nginx:${VERSION}
ARG VERSION
RUN echo ${VERSION}
```
以下构建输出在第四步显示 echo 值

```shell
[root@node1 docker]# docker build --build-arg VERSION=alpine .
Sending build context to Docker daemon  2.048kB
Step 1/4 : ARG VERSION
Step 2/4 : FROM nginx:${VERSION}
alpine: Pulling from library/nginx
bdf0201b3a05: Already exists 
08d74e155349: Pull complete 
a9e2a0b35060: Pull complete 
d9e2304ab8d0: Pull complete 
Digest: sha256:61e3db30b1334b1fa0a2e71b86625188f76653565d515d5f74ecc55a8fb91ce9
Status: Downloaded newer image for nginx:alpine
 ---> 0be75340bd9b
Step 3/4 : ARG VERSION
 ---> Running in 57b9d8bccc02
Removing intermediate container 57b9d8bccc02
 ---> d7b120dcb6cc
Step 4/4 : RUN echo ${VERSION}
 ---> Running in db84122a22b4
alpine              ###############    注意这！！！！！
Removing intermediate container db84122a22b4
 ---> eb8b05d72b74
Successfully built eb8b05d72b74

```

下面示例是只在 FROM 前使用 `ARG 指令`

```yaml
ARG VERSION
FROM nginx:${VERSION}
RUN echo ${VERSION:-1.0}
```
以下输出中，FROM引用了 ARG 的值，FROM之后的RUN指令未引用，而是使用了我们规定的默认值。

```shell
# docker build --build-arg VERSION=latest .
Sending build context to Docker daemon  2.048kB
Step 1/3 : ARG VERSION
Step 2/3 : FROM nginx:${VERSION}
latest: Pulling from library/nginx
27833a3ba0a5: Pull complete 
ea005e36e544: Pull complete 
d172c7f0578d: Pull complete 
Digest: sha256:e71b1bf4281f25533cf15e6e5f9be4dac74d2328152edf7ecde23abc54e16c1c
Status: Downloaded newer image for nginx:latest
 ---> 27a188018e18
Step 3/3 : RUN echo ${VERSION:-1.0}
 ---> Running in 856022e6f98d
1.0           ################ 注意这里的值！！！
Removing intermediate container 856022e6f98d
 ---> 69d12f7d9648
Successfully built 69d12f7d9648

```

**ARG 默认值的设定**

在构建时，如果没有传递值，则使用默认值

```yaml
FROM busybox
ARG user1=someuser
ARG buildno=1
```

## VOLUME 挂载卷

在 Dockerfile 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。

```yaml
VOLUME /data
```

## EXPOSE   申明端口

`EXPOSE` 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。在 Dockerfile 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 `docker run -P` 时，会自动随机映射 `EXPOSE` 的端口。

您可以指定端口是侦听TCP还是UDP，如果未指定协议，则默认为TCP。

无论EXPOSE设置如何，都可以使用` -p `标志在运行时覆盖它们

## USER 指定当前用户

USER 指令和 WORKDIR 相似，都是改变环境状态并影响以后的层。WORKDIR 是改变工作目录，USER 则是改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类命令的身份。

当然，和 WORKDIR 一样，USER 只是帮助你切换到指定用户而已，这个用户必须是事先建立好的，否则无法切换。

```yaml
RUN groupadd -r redis && useradd -r -g redis redis
USER redis
RUN [ "redis-server" ]
```

如果以 root 执行的脚本，在执行期间希望改变身份，比如希望以某个已经建立好的用户来运行某个服务进程，不要使用 su 或者 sudo，这些都需要比较麻烦的配置，而且在 TTY 缺失的环境下经常出错。建议使用 [gosu](https://github.com/tianon/gosu)。

```yaml
# 建立 redis 用户，并使用 gosu 换另一个用户执行命令
RUN groupadd -r redis && useradd -r -g redis redis
# 下载 gosu
RUN wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/1.7/gosu-amd64" \
    && chmod +x /usr/local/bin/gosu \
    && gosu nobody true
# 设置 CMD，并以另外的用户执行
CMD [ "exec", "gosu", "redis", "redis-server" ]
```

## HEALTHCHECK   健康检查

`HEALTHCHECK` 指令是告诉 Docker 应该如何进行判断容器的状态是否正常，这是 Docker 1.12 引入的新指令。

格式：
```
HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令
```

在一个镜像指定了 HEALTHCHECK 指令后，用其启动容器，初始状态会为 starting，在 HEALTHCHECK 指令检查成功后变为 healthy，如果连续一定次数失败，则会变为 unhealthy。

HEALTHCHECK 支持下列选项：

--interval=<间隔>：两次健康检查的间隔，默认为 30 秒；
--timeout=<时长>：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
--retries=<次数>：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。
和 CMD, ENTRYPOINT 一样，HEALTHCHECK 只可以出现一次，如果写了多个，只有最后一个生效。

在 HEALTHCHECK [选项] CMD 后面的命令，格式和 ENTRYPOINT 一样，分为 shell 格式，和 exec 格式。命令的返回值决定了该次健康检查的成功与否：0：成功；1：失败；2：保留

## ONBUILD  

格式：ONBUILD <其它指令>。

ONBUILD 是一个特殊的指令，它后面跟的是其它指令，比如 RUN, COPY 等，而这些指令，在当前镜像构建时并不会被执行。只有当以当前镜像为基础镜像，去构建下一级镜像的时候才会被执行。

Dockerfile 中的其它指令都是为了定制当前镜像而准备的，唯有 ONBUILD 是为了帮助别人定制自己而准备的。

```YAML
FROM node:slim
RUN mkdir /app
WORKDIR /app
ONBUILD COPY ./package.json /app
ONBUILD RUN [ "npm", "install" ]
ONBUILD COPY . /app/
CMD [ "npm", "start" ]
```

这是一个很好的示例。


## CMD 和 ENTRYPOINT

`CMD`和`ENTRYPOINT` 指令都是用来指定容器运行时的命令

单从功能上看，在dockerfile中使用两者之一区别不大，但是既然有两个指令肯定就有什么猫咪。

想要彻底理解这两个命令为什么会同时存在，就需要先了解`exec模式`和`shell模式`，这两种模式涉及到基础知识-- `sh` `source` `exec` 执行脚本有什么区别

**sh**

当前shell是父进程，生成一个子shell进程，在子shell中执行脚本。脚本执行完毕，退出子shell，回到当前shell。

`sh xx.sh` 等同于 `./xx.sh`

```shell
#!/bin/sh
while true;do
    echo $$
    sleep 1
done
```
`sh process.sh` 使用 `htop` 查看进程树

![20190419115402](https://img.24linux.com/static/images/20190419115402.png %}

父进程 16130 生成子进程 16528 

**source**

在当前上下文中执行脚本，不会生成新的进程。脚本执行完毕，回到当前shell。

`source xx.sh` 等同于 `. xx.sh`

```shell
# source process.sh 
16130
16130
16130
16130
16130
```

**exec**
 
使用exec command方式，会用command进程替换当前shell进程，并且保持PID不变。执行完毕，直接退出，不回到之前的shell环境。

```shell
[root@VM_79_16_centos ~]# echo $$
18859
[root@VM_79_16_centos ~]# exec echo $$
18859
Connection closing...Socket close.

```
当前shell的进程ID 18859 ，使用exec 执行命令后，当前shell退出，不回到之前的shell环境。

**使用 sh 或者 source 对上下文的影响**

```shell
#!/bin/sh
cd /
pwd
echo hello
```

使用 `sh` 命令执行

![20190419134944](https://img.24linux.com/static/images/20190419134944.png)

执行完后，还是在家目录，并未受到 `cd /` 的影响

使用 `source` 命令执行

![20190419135157](https://img.24linux.com/static/images/20190419135157.png)

执行完后，受到 `cd /` 命令的影响。

通过source执行脚本时，修改的上下文会影响当前shell；通过sh执行脚本时，修改的上下文不会影响当前shell。

CMD 和 ENTRYPOINT 指令都支持 exec 模式和 shell 模式的写法，所以要理解 CMD 和 ENTRYPOINT 指令的用法，就得先区分 exec 模式和 shell 模式。这两种模式主要用来指定容器中不同的进程为 1 号进程。

## 现在已经知道了 exec 模式和 shell 模式 的区别(exec 以主进程的方式运行程序，shell 由当前shell fork子进程的方式运行程序) 也就是 exec模式运行的容器进程是pid为`1的进程` ，而 shell模式运行的容器进程pid并不是1的进程。

<font color=red>为啥容器内的程序最好要是以进程 pid 为 1的方式运行？</font>

Docker 不是虚拟机，容器就是进程，在linux系统中，1号进程的作用不言而喻，如下图

![20190419141723](https://img.24linux.com/static/images/20190419141723.png)

关于systemd对系统的管理，可以学习下[阮一峰老师的教程](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)。

具体得大家可以看看另外一篇文章[《在docker中捕捉信号》](http://www.cnblogs.com/sparkdev/p/7598590.html) ；

Docker 的 stop 和 kill 命令都是用来向容器发送信号的。注意，只有容器中的 1 号进程能够收到信号，这一点非常关键！

## CMD  

shell 格式：CMD <命令>

exec 格式：CMD ["可执行文件", "参数1", "参数2"...]

参数列表格式：CMD ["参数1", "参数2"...]。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。

dockerfile中只能有一个 CMD 指令，如果有多个，只会生效最后一个指令。
 
注意命令行参数可以覆盖 CMD 指令的设置，但是只能是重写，却不能给 CMD 中的命令通过命令行传递参数。

一般的镜像都会提供容器启动时的默认命令，但是有些场景中用户并不想执行默认的命令。用户可以通过命令行参数的方式覆盖 CMD 指令提供的默认命令。如下：

**使用alpine为基础镜像，并以镜像中的默认命令启动：**

```shell
# cat Dockerfile 
FROM alpine

# docker build .
Sending build context to Docker daemon  2.048kB
Step 1/1 : FROM alpine
latest: Pulling from library/alpine
bdf0201b3a05: Already exists 
Digest: sha256:28ef97b8686a0b5399129e9b763d5b7e5ff03576aa5580d6f4182a49c5fe1913
Status: Downloaded newer image for alpine:latest
 ---> cdf98d1859c1
Successfully built cdf98d1859c1

# docker run -it cdf
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    5 root      0:00 ps -ef
/ # 
```

**使用alpine为基础镜像，并用CMD指令提供的默认命令启动：**

```shell
# cat Dockerfile 
FROM alpine
CMD ["top"]

# docker run -d cb
500a1a8d02181d176bb8d4cab1241d6c46b0d0959ade2416f8b72c19739a73d7

# docker exec -it 500 ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 top
    5 root      0:00 ps -ef
```

使用CMD命令指定默认命令启动，容器启动后，查看容器得进程id如上。使用了CMD命令指定的`top`命令作为1号进程。

**使用apline为基础镜像，并用CMD指令提供的默认命令启动，在启动容器时使用命令行参数覆盖dockerfile中指定的CMD指令：**

```shell
# cat Dockerfile 
FROM alpine
CMD ["top"]

# docker run --rm cb ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 ps -ef

```
dockerfile中CMD指定的命令被覆盖。

## ENTRYPOINT

ENTRYPOINT 的格式和 RUN 指令格式一样，分为 exec 格式和 shell 格式。

ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数。ENTRYPOINT 在运行时也可以替代，不过比 CMD 要略显繁琐，需要通过 docker run 的参数 --entrypoint 来指定。

当指定了 ENTRYPOINT 后，CMD 的含义就发生了改变，不再是直接的运行其命令，而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令，换句话说实际执行时，将变为：

<ENTRYPOINT> "<CMD>"

**以alpine为基础镜像，安装 curl包**

```yaml
FROM alpine
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories \
    && apk update \
    && apk add --no-cache curl \
    && rm -rf /var/cache/apk/* 
ENTRYPOINT ["curl","-s","https://ip.cn"]

```
```shell
# docker run --rm 524
当前 IP: 193.112.95.105 来自: 广东省广州市 腾讯
```

如果我们需要查看请求头信息，命令为 `curl -s -i https://ip.cn` ，那如何将 `-i`参数传进去呢？


CMD 指令可以覆盖没错， ENTRYPOINT命令也可以覆盖也没错，但是有没有更简单的，只需要传个参数进去的方法呢？ -有

```shell
# docker run --rm b2e -i
HTTP/2 200 
date: Fri, 19 Apr 2019 08:02:12 GMT
content-type: text/html; charset=UTF-8
set-cookie: __cfduid=dde6679fd0958873908cf4fc4872f9b451555660932; expires=Sat, 18-Apr-20 08:02:12 GMT; path=/; domain=.ip.cn; HttpOnly
expect-ct: max-age=604800, report-uri="https://report-uri.cloudflare.com/cdn-cgi/beacon/expect-ct"
server: cloudflare
cf-ray: 4c9d5adc1a486d66-SJC

当前 IP: 193.112.95.105 来自: 广东省广州市 腾讯

```
如果dockerfile中使用的不是 `ENTRYPOINT` 而是 `CMD` 那么就需要在命令行传递整个命令，而`ENTRYPOINT` 可以将`CMD`传递的值当作参数使用。

最后附上官方 CMD && ENTRYPOINT 的协作图

![20190419143029.png](https://img.24linux.com/static/images/20190419143029.png)

---
参考

https://www.cnblogs.com/sparkdev/p/8461576.html
https://yeasy.gitbooks.io/docker_practice/content/image/dockerfile/entrypoint.html
https://docs.docker.com/engine/reference/builder/
https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
