# Swarm 基本概念及使用

## swarm功能亮点
1. swarm与docker引擎集成，无需安装，直接使用
2. 分布式设计
3. 声明式服务模型
4. 扩缩容
5. status状态期望
6. 服务发现
7. 多主机网络
8. service负载均衡
9. 安全性高
10. 滚动动态更新

Swarm 是使用 SwarmKit 构建的 Docker 引擎内置（原生）的集群管理和编排工具。

使用 Swarm 集群之前需要了解以下几个概念。

## 节点
运行 Docker 的主机可以主动初始化一个 Swarm 集群或者加入一个已存在的 Swarm 集群，这样这个运行 Docker 的主机就成为一个 Swarm 集群的节点 (node) 。

节点分为管理 (manager) 节点和工作 (worker) 节点。

管理节点用于 Swarm 集群的管理，docker swarm 命令基本只能在管理节点执行（节点退出集群命令 docker swarm leave 可以在工作节点执行）。一个 Swarm 集群可以有多个管理节点，但只有一个管理节点可以成为 leader，leader 通过 raft 协议实现。

工作节点是任务执行节点，管理节点将服务 (service) 下发至工作节点执行。管理节点默认也作为工作节点。你也可以通过配置让服务只运行在管理节点。
![](https://img.24linux.com/static/images/20190423154529.png)

## 服务和任务
任务 （Task）是 Swarm 中的最小的调度单位，目前来说就是一个单一的容器。

服务 （Services） 是指一组任务的集合，服务定义了任务的属性。服务有两种模式：

replicated services 按照一定规则在各个工作节点上运行指定个数的任务。

global services 每个工作节点上运行一个任务

两种模式通过 docker service create 的 --mode 参数指定。

![](https://img.24linux.com/static/images/20190423154548.png)

## 开始使用
我学习swarm的环境： 使用vmware安装的三台虚拟机  

version : Linux node1 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux

node1 : 192.168.18.11  (master)
node2 : 192.168.18.12  (worker1)
node3 : 192.168.18.13  (worker2)

docker version :

```
Client:
 Version:           18.09.0
 API version:       1.39
 Go version:        go1.10.4
 Git commit:        4d60db4
 Built:             Wed Nov  7 00:48:22 2018
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.0
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.4
  Git commit:       4d60db4
  Built:            Wed Nov  7 00:19:08 2018
  OS/Arch:          linux/amd64
  Experimental:     false

```

ports:

2377    用于集群管理通信

7946    节点通信 

## 创建一个swarm集群

`docker swarm init --advertise-addr <MANAGER-IP>`

`--advertise-addr` 管理节点将MANAGER-IP作为发布地址，加入集群的节点必须能访问到该地址。


使用 `docker info` 查看swarm当前状态：

```shell
Containers: 8
 Running: 1
 Paused: 0
 Stopped: 7
Images: 19
...
...
Swarm: active
 NodeID: d3ji5fpl2c9adbequo47na66f
 Is Manager: true
 ClusterID: f6wqn9uoht2215kw4h3r5y9zf
 Managers: 1
 Nodes: 3
 Default Address Pool: 10.0.0.0/8  

```
使用 `docker node ls`查看节点信息

```shell
# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
d3ji5fpl2c9adbequo47na66f *   node1               Ready               Active              Leader              18.09.0

```

## 将节点添加到集群中

### 作为工作节点worker加入集群

`docker swarm join-token worker `

```shell
# docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-31vozn9xpbit6u8qkutpdq2u29g93e37uuy7zja5ps2qkcj6kp-9myq79lv63hzy65ionbagva3w 192.168.18.11:2377

```

### 作为管理节点manager加入集群

`docker swarm join-token manager`

```shell
# docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-31vozn9xpbit6u8qkutpdq2u29g93e37uuy7zja5ps2qkcj6kp-33tocce2c2gnz2vfkjixx52xd 192.168.18.11:2377

```

### docker swarm join 会执行哪些操作

1. 将该节点的docker Engine 切换到 swarm模式
2. 想master请求 TLS证书
3. 使用该机主机名作为节点名称
4. 使用token将当前节点连接到master管理节点
5. 将当前节点设置为`Active`，表示可以接受任务调度
6. 将ingress网络覆盖到当前节点

## 管理集群中的节点

### 查看节点

```shell
# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
38toev7jiw6q7kk2m0fvisq5h     master              Ready               Active              Reachable           18.09.0
d3ji5fpl2c9adbequo47na66f *   node1               Ready               Active              Leader              18.09.0
98wsuytibxtm8lia7md1jauaw     node2               Ready               Active                                  18.09.0
vf8fk88tlz5436xw5yw65lrfl     node3               Ready               Active                                  18.09.0
```

`AVAILABILITY` 列显示调度程序是否可以将任务分配给节点：

`Active` 表示调度程序可以将任务分配给节点。
`Pause` 表示调度程序不会将新任务分配给节点，但现有任务仍在运行。
`Drain` 表示调度程序不会将新任务分配给节点。调度程序关闭所有现有任务并在可用节点上调度它们。


`MANAGER STATUS` 列显示manager状态：

没有值表示不参与群组管理的工作节点。
`Leader` 表示节点是主管理器节点，它为群集做出所有群集管理和编排决策。
`Reachable` 表示该节点是参与Raft共识仲裁的管理者节点。如果主节点变得不可用，则该节点有资格被选为主节点。
`Unavailable` 表示节点是无法与其他管理器通信的管理节点。如果管理节点变得不可用，您应该将新的管理节点加入到群组中，或者将工作节点提升为管理节点。

### 单个节点的详细信息

```shell
# docker node inspect --pretty vf
ID:			vf8fk88tlz5436xw5yw65lrfl
Hostname:              	node3
Joined at:             	2019-04-23 18:30:33.496632341 +0000 utc
Status:
 State:			Ready
 Availability:         	Active
 Address:		192.168.18.13

```

`--pretty ` 使用该参数，不会以json格式输出

### 更新节点

节点维护时，可以使用该命令。

node2设置为`Drain`状态后，该节点上的程序会调度到其他节点上。

```
# docker node update --availability drain node2
node2
```

### 提升或降低级别
`docker node promote NODE-ID`  提升为manager

```shell
# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
kl5bizcmbbhxt0p8egrsoi2wg     master              Ready               Active                                  18.09.0
nr0zumcakvid6nup95frqvzpw *   node1               Ready               Active              Leader              18.09.0
3w5hsx8fybnvkxgahcy4wtv5a     node2               Ready               Active                                  18.09.0
lo6uktnniir3ihn13lz39xuek     node3               Ready               Active                                  18.09.0

[root@node1 ~]# docker node promote master
Node master promoted to a manager in the swarm.

[root@node1 ~]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
kl5bizcmbbhxt0p8egrsoi2wg     master              Ready               Active              Reachable           18.09.0
nr0zumcakvid6nup95frqvzpw *   node1               Ready               Active              Leader              18.09.0
3w5hsx8fybnvkxgahcy4wtv5a     node2               Ready               Active                                  18.09.0
lo6uktnniir3ihn13lz39xuek     node3               Ready               Active                                  18.09.0

```

`docker node demote NODE-ID` 降级为worker

使用 `docker node update --role` 也能实现同样的效果

```shell
# docker node update -h
Flag shorthand -h has been deprecated, please use --help

Usage:	docker node update [OPTIONS] NODE

Update a node

Options:
      --availability string   Availability of the node ("active"|"pause"|"drain")
      --label-add list        Add or update a node label (key=value)
      --label-rm list         Remove a node label if exists
      --role string           Role of the node ("worker"|"manager")

```

### 给节点添加标签
`docker node update --label-add `

### 离开节点
在worker节点上使用命令 `docker swarm leave ` 可以脱离集群，但是在manager上还是会显示节点的存在。这时可以在manager节点使用 `docker node rm` 删除已经退出的节点

## 部署服务

### 创建服务

```shell
# docker service create --name myweb nginx
stzte0t7sifueeiqd2d3wn75w
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 
```

### 使用私有仓库的镜像创建服务
单独将私有仓库创建服务做笔记是因为当初最开始学的时候不知道这个参数，一直在疑惑一个问题，swarm是编排集群上的容器的，如果我自己使用dockerfile创建了image，上传到私有仓库，在使用`docker service create`创建服务时，大部分情况下你不会去指定服务容器创建在哪台宿主机上，假如创建在A主机上，那么A主机需要从私有仓库拉取你build创建的镜像，这就需要登录，那假如在B主机呢，你还得去B登录！这就不行了，所以 `--with-registry-auth` 参数了解一下。

现在我先讲准备好的镜像推送到自己的私服仓库，这里使用阿里云的镜像仓库，还是挺好用的，图中的k8s镜像是我学习的时候创建的，因为这些镜像因为不可描述的原因无法直接下载，通过阿里云容器镜像仓库创建后pull到本地，再tag。

![](https://img.24linux.com/static/images/20190426110731.png)

```shell
# docker login --username=mumengyun registry.cn-hangzhou.aliyuncs.com
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@node1 ~]# docker push registry.cn-hangzhou.aliyuncs.com/mmyregistry/myapp:v3
The push refers to repository [registry.cn-hangzhou.aliyuncs.com/mmyregistry/myapp]
e9247e33ca13: Pushed 
05a9e65e2d53: Pushed 
68695a6cfd7d: Pushed 
c1dc81a64903: Pushed 
8460a579ab63: Pushed 
d39d92664027: Pushed 
v3: digest: sha256:fbf10a2b3490806a85badab94916e46623d65494b3573821fb57190f07afdabc size: 1569
```

以上终端输出中，*WARNING! Your password will be stored unencrypted in /root/.docker/config.json.*  意思是你的密码以未加密的形式保存在这个文件中，我们查看一下

```shell
[root@node1 ~]# cat /root/.docker/config.json
{
	"auths": {
		"registry.cn-hangzhou.aliyuncs.com": {
			"auth": "xxxxxxxxxxxxxxxxxxxxxxxxx"
		}
	},
	"HttpHeaders": {
		"User-Agent": "Docker-Client/18.09.0 (linux)"
	}
```

接下来，我们使用这个私有镜像创建服务，并且启动4个副本。 注意：这时候只有一台主机上登陆了私服账户

![](https://img.24linux.com/static/images/20190426112149.png)

显示未找到镜像，因为我们未加 `--with-registry-auth` 参数

![](https://img.24linux.com/static/images/20190426114119.png)

部署成功！其他节点下载镜像也没有问题。

## 服务创建后更新

使用 `docker service update -h ` 可以查看支持的更新参数。太多了，这里只记录image的滚动更新，回滚等笔记。

![](https://img.24linux.com/static/images/20190426115308.png)

上节中使用的镜像版本是 `v3` 现在我们创建一个 `v2` 版本的镜像，并上传到阿里云容器镜像仓库私服。

![](https://img.24linux.com/static/images/20190426115556.png)

使用 `v2` 版本更新旧 `v3` 版本。

![](https://img.24linux.com/static/images/20190426120055.png)

![](https://img.24linux.com/static/images/20190426120202.png)

更新镜像成功，并且记录了更新后的信息，同时保留了更新前的信息。

## 服务回滚

`docker service rollback ` 命令进行回退操作

![](https://img.24linux.com/static/images/20190426120515.png)

不管是滚动更新，回退等操作，都可以添加更详细的参数，比如，滚动更新时间间隔，更新失败后的操作等。往往这些参数会写在 comeposefile 文件中。
