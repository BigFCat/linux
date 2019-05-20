# 镜像image操作命令

## 拉取镜像 docker pull 

注意，未指明版本时默认拉取latest版本

```shell
# docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
fc1a6b909f82: Pull complete 
Digest: sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd
Status: Downloaded newer image for busybox:latest
```
## 查看镜像 docker images
上图中[官方镜像](https://hub.docker.com/_/busybox?tab=tags)中显示`busybox`这个镜像只有756K，实际下载下来1.2M，这是为啥？

Docker Hub 中显示的是压缩后的体积。在镜像下载和上传过程中镜像是保持着压缩状态的，因此 Docker Hub 所显示的大小是网络传输中的流量大小。而 `docker image ls` 显示的是镜像下载到本地后，展开的大小，准确说，是展开后的各层所占空间的总和，因为镜像到本地后，查看空间的时候，查看是本地磁盘空间占用的大小。

但是，`dockers images ls` 查看的所有镜像的大小的总和并不一定是实际占用磁盘的大小，doker镜像采用的分层存储结构，其他镜像有可能继承复用同一基础镜像的其他镜像。由于 Docker 使用 Union FS，相同的层只需要保存一份即可，因此实际镜像硬盘占用空间很可能要比这个列表镜像大小的总和要小的多。

```shell
# docker images busybox
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              af2f74c517aa        2 weeks ago         1.2MB
```

##  以特定格式显示image镜像列表
1. 只显示镜像id  `-q`参数
```shell
# docker images -q
93e95052b8a0
6f2f8a0a93ad
7a4dc0e05ef9
```
2. 使用GO模板输出指定内容
[GO模板官方链接](https://gohugo.io/templates/introduction/)
```
#docker image ls --format "\{\{.ID}}: \{\{.Repository}}"
af2f74c517aa: busybox
5cd54e388aba: k8s.gcr.io/kube-proxy
```

## 删除镜像 docker rmi
```
# docker images --help

Usage:	docker images [OPTIONS] [REPOSITORY[:TAG]]

List images

Options:
  -a, --all             Show all images (default hides intermediate images)
      --digests         Show digests
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print images using a Go template
      --no-trunc        Don't truncate output
  -q, --quiet           Only show numeric IDs
```

根据帮助文档可以看出还可以显示digests

```shell
# docker images --digests
REPOSITORY          TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
busybox             latest              sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd   af2f74c517aa        2 weeks ago         1.2MB
```

删除镜像命令 `docker rmi` ，删除镜像时，对象可以输入`镜像长ID`，也可以是`短ID(必须唯一)`，或者`名称:tag`或者`digest`。

```shell
# docker rmi busybox@sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd
Untagged: busybox@sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd
```

这里有一个问题，`Untagged`和`Deleted`

```shell
# docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
27833a3ba0a5: Pull complete 
ea005e36e544: Pull complete 
d172c7f0578d: Pull complete 
Digest: sha256:ad250db8c3c15a6d4ee51a9e6947ca16a1010bf958c9db1d7223450ac88e1f28
Status: Downloaded newer image for nginx:latest

# docker images nginx
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              27a188018e18        11 hours ago        109MB

# docker rmi 27a1
Untagged: nginx:latest
Untagged: nginx@sha256:ad250db8c3c15a6d4ee51a9e6947ca16a1010bf958c9db1d7223450ac88e1f28
Deleted: sha256:27a188018e1847b312022b02146bb7ac3da54e96fab838b7db9f102c8c3dd778
Deleted: sha256:261d1996088c57b71d8ea9412f719bcbb8f4cb68a6e463d30abb85cc5fc5724b
Deleted: sha256:e6fbd1f039a7391ab57afeb1b11a73781bcbd6ae8041d98c5988b90c46ce5726
Deleted: sha256:5dacd731af1b0386ead06c8b1feff9f65d9e0bdfec032d2cd0bc03690698feda
```

上面示例中，我先pull获取了nginx镜像，然后删除该镜像，删除的时候有两个状态，这是为啥？

镜像的唯一标识是其 ID 和摘要，而一个镜像可以有多个标签。

因此当我们使用上面命令删除镜像的时候，实际上是在要求删除某个标签的镜像。所以首先需要做的是将满足我们要求的所有镜像标签都取消，这就是我们看到的 Untagged 的信息。因为一个镜像可以对应多个标签，因此当我们删除了所指定的标签后，可能还有别的标签指向了这个镜像，如果是这种情况，那么 Delete 行为就不会发生。所以并非所有的 `docker image rm` 都会产生删除镜像的行为，有可能仅仅是取消了某个标签而已。

当该镜像所有的标签都被取消了，该镜像很可能会失去了存在的意义，因此会触发删除行为。镜像是多层存储结构，因此在删除的时候也是从上层向基础层方向依次进行判断删除。镜像的多层结构让镜像复用变动非常容易，因此很有可能某个其它镜像正依赖于当前镜像的某一层。这种情况，依旧不会触发删除该层的行为。直到没有任何层依赖当前层时，才会真实的删除当前层。这就是为什么，有时候会奇怪，为什么明明没有别的标签指向这个镜像，但是它还是存在的原因，也是为什么有时候会发现所删除的层数和自己 `docker pull` 看到的层数不一样的源。

除了镜像依赖以外，还需要注意的是容器对镜像的依赖。如果有用这个镜像启动的容器存在（即使容器没有运行），那么同样不可以删除这个镜像。之前讲过，容器是以镜像为基础，再加一层容器存储层，组成这样的多层存储结构去运行的。因此该镜像如果被这个容器所依赖的，那么删除必然会导致故障。如果这些容器是不需要的，应该先将它们删除，然后再来删除镜像。

`--filter` 参数：

[官方过滤参考文档](https://docs.docker.com/engine/reference/commandline/images/)  `-f`是很有用的参数，往往在管理镜像和容器的时候能发挥出奇效，可以参考官方文档使用

## 查看镜像详细信息 docker inspect 
`docker inspect` 命令返回的一个jsons格式的数组，官方提供了使用GO模板获取指定信息的方法。在 Linux系统中也可以通过安装`jq`包，来获取返回的json格式数据中的某个值。

`docker inspect xx` 返回的是一个json格式的数据

```
[
    {
        "Id": "706813b0da107c4d43c61e3db9da90847fbb8ffcdb5f3022696ac997",
        "Created": "2018-11-28T10:06:08.081429716Z",
        "Path": "java",
        "Args": [
            "-Xmx1000m",
            "-jar",
            "/app/xxxx.jar"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 16567,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2018-11-28T10:06:08.445847694Z",
            "FinishedAt": "0001-01-01T00:00:00Z",
            "Health": {
                "Status": "healthy",
                "FailingStreak": 0,
                "Log": [
                    {
                        "Start": "2018-11-29T11:35:01.911518247+08:00",
                        "End": "2018-11-29T11:35:02.064646904+08:00",
                        "ExitCode": 0,
...
...
```

**获取容器的ip的第一种方法:**
```
# docker inspect 7a | jq -r '.[0].NetworkSettings.Networks.elk_elk.IPAddress'
172.20.0.2
```
`.[0]` 取数组的第一个值,如下图，inspect返回的是一个字典的数组，`.[0]`就是取第一个元素，也就是整个返回值

![201904171642](https://img.24linux.com/static/images/201904171642.png)
后面的就是正常的json取值方式，方便很多！

**获取容器的ip的第二种方法:**
```
# docker inspect 7a --format {{.NetworkSettings.Networks.elk_elk.IPAddress}}
172.20.0.2
```
>Docker CLI 的 --format 参数提供了基于 Go模板 的日志格式化输出辅助功能，并提供了一些内置的增强函数。

>Go语言提供了简单灵活的模板支持，而基于 Go 开发的 Docker 继承了该强大能力，使其可以脱离 Shell 的相关操作，直接对结果进行格式化输出。所有支持 --format 扩展的 Docker CLI 指令均支持该操作。

 Go模板格式化输出推荐文章：https://yq.aliyun.com/articles/230067

## docker commit
这个命令正常情况不推荐使用，在某些特殊场景下会用到，比如容器被入侵，保存镜像现场。