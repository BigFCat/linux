# compose 模板文件构建

## build 

每个服务必须通过`image`指令或者`build`指令(需要Dockerfile)等来自动构建生成镜像

```yaml
version: "3"
services:
  webapp:
    build: 
      context: .
      dockerfile: Dockerfile.nginx
      args:
        buildno: 1 
```

### context

指定相对于 docker-compose.yaml 文件的路径的 Dockerfile的目录，上面yaml中，Dockerfile.nginx 和 docker-compose.yaml 就在同一层级中。也可以是绝对路径。

### dockerfile

指定 dockerfile 的文件名

### args

指定构建镜像时候的变量，常与 Dockerfile 中ARG指令结合使用.

布尔值 （true，false，yes，no，on，off） 必须用引号括起来

### cache_from

镜像缓存

### labels

添加标签给镜像，建议使用反向DNS表示法

```yaml
build:
  context: .
  labels:
    com.example.description: "Accounting webapp"
    com.example.department: "Finance"
    com.example.label-with-empty-value: ""
```

### shm_size



### trget

## image 指定镜像

build 中可以指定镜像，但是是基于Dockerfile的，build.image也是指定镜像构建后的名称，在顶层层级下使用 image 才是指定构建镜像。

## cap_add  cap_drop



## cgroup_parent

## command

覆盖默认命令

## config

参照[Swarm集群管理配置信息configs](/swarm/config.md)

### Short syntax

```yaml
version: "3.7"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - my_config
      - my_other_config
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```

### Long syntax

```yaml
version: "3.7"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - source: my_config
        target: /redis_config
        uid: '103'
        gid: '103'
        mode: 0440
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```

## container_name

容器名称

## credential_spec

此项仅用于Windows容器服务

## depends_on 

控制启动顺序

下面示例中redi和db服务会先启动，再启动web服务，但是web服务并不是在redis和db服务启动并且能接收请求后才会启动。

swarm模式下会忽略该选项

```yaml
version: "3.7"
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```

## deploy 

deploy是用来指定swarm服务部署和运行时的相关配置，并且只有使用docker stack deploy 部署swarm集群时才会生效。如果使用docker-compose up 或者docker-compose run时，该选项会被忽略。要使用deploy选项，compose-file中version版本要在3或3+

### endpoint_mode 服务发现模式

指定swarm服务发现的模式

endpoint_mode: vip 
- Docker为swarm集群服务分配一个虚拟IP(VIP)，作为客户端到达集群服务的“前端”。Docker 在客户端和可用工作节点之间对服务的请求进行路由。而客户端不用知道有多少节点参与服务或者是这些节点的IP/端口。（这是默认模式）
  
endpoint_mode: dnsrr 
- DNS轮询（DNSRR）服务发现不使用单个虚拟IP。 Docker为服务设置DNS条目，使得服务名称的DNS查询返回一个IP地址列表，并且客户端直接连接到其中的一个。如果您想使用自己的负载平衡器，或者混合Windows和Linux应用程序，则DNS轮询功能非常有用。

```yaml
version: "3.7"

services:
  wordpress:
    image: wordpress
    ports:
      - "8080:80"
    networks:
      - overlay
    deploy:
      mode: replicated
      replicas: 2
      endpoint_mode: vip

  mysql:
    image: mysql
    volumes:
       - db-data:/var/lib/mysql/data
    networks:
       - overlay
    deploy:
      mode: replicated
      replicas: 2
      endpoint_mode: dnsrr

volumes:
  db-data:

networks:
  overlay:
```

### labels 

服务标签，不是容器标签，容器标签在build选项中配置

### mode 副本模式

`global` 和 `replicated` 两种模式。 `global模式` 每个节点一个副本容器；`replicated模式` 副本模式，可以指定容器个数，系统将调度到合适的节点。

### placement 约束

指定服务到更具体的节点

```yaml
version: "3.7"
services:
  db:
    image: postgres
    deploy:
      placement:
        constraints:
          - node.role == manager
          - engine.labels.operatingsystem == ubuntu 14.04
        preferences:
          - spread: node.labels.zone
```

### constraints 约束项

![](https://img.24linux.com/static/images/20190428182507.png)

### preferences 首选项

[参考docker service create](https://docs.docker.com/engine/reference/commandline/service_create/#specify-service-placement-preferences---placement-pref)

### replicas  副本

```yaml
version: "3.7"
services:
  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 6
```

### resources  资源限制

配置资源限制

```yaml
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 50M
        reservations:
          cpus: '0.25'
          memory: 20M
```

## restart_policy 容器退出时的重启策略

```yaml
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```

`condition`  

`deploy` 容器重启前等待的时间，默认为0

`max_attempts` 最大重启次数，默认无限

`window` 在判断是否成功重启时，等待多长时间。 

后面两个参数没有弄太清楚！ 两个参数需要结合使用。 [官方文档重启策略](https://docs.docker.com/compose/compose-file/#restart_policy)

## rollback_config  更新失败回滚策略

`parallelism`：一次回滚的容器数。如果设置为0，则所有容器同时回滚。
`delay`：每个容器组的回滚之间等待的时间（默认为0）。
`failure_action`：如果回滚失败该怎么办。一个continue或pause（默认pause）
`monitor`：每次更新任务后的持续时间以监视失败(ns|us|ms|s|m|h)（默认为0）。
`max_failure_ratio`：回滚期间容忍的失败率（默认为0）。
`order`：回滚期间的操作顺序。其中一个stop-first（旧任务在启动新任务之前停止），或者start-first（首先启动新任务，并且正在运行的任务暂时重叠）（默认stop-first）。


## update_config  服务更新策略

`parallelism`：一次更新的容器数。
`delay`：更新一组容器之间的等待时间。
`failure_action`：如果更新失败该怎么办。其中一个continue，rollback或者pause （默认：pause）。
`monitor`：每次更新任务后的持续时间以监视失败(ns|us|ms|s|m|h)（默认为0）。
`max_failure_ratio`：更新期间容忍的失败率。
`order`：更新期间的操作顺序。其中一个stop-first（旧任务在启动新任务之前停止），或者start-first（新任务首先启动，运行任务暂时重叠）（默认stop-first）注意：仅支持v3.4及更高版本。

## devices 设备映射

```yaml
devices:
  - "/dev/ttyUSB0:/dev/ttyUSB0"
```

## DNS

## dns_search

## entrypoint

## env_file

从文件添加环境变量，可以是单个值或者列表

如果已经指定Compose文件，`docker-compose -f FILE` 则 env_file 文件的相对路径相对于compose文件目录

```yaml
env_file: .env

或者

env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```

Compose期望env文件中的每一行都是VAR=VAL格式化的。空行被忽略。

```yaml
# Set Rails/Rack environment
RACK_ENV=development
```

## environment 环境变量

```yaml
environment:
  RACK_ENV: development
  SHOW: 'true'
  SESSION_SECRET:

或者

environment:
  - RACK_ENV=development
  - SHOW=true
  - SESSION_SECRET
```

## expose 端口暴露

仅仅是容器内部端口暴露，不是端口映射到主机

```yaml
expose:
 - "3000"
 - "8000"
```

## external_links 外部链接

一般docker-compose.yaml文件中编排的容器都在自定义的一个网络中，但是如果要链接该网络外的容器，就需要使用该指令。

```yaml
external_links:
 - redis_1
 - project_db_1:mysql
 - project_db_1:postgresql
```

## extra_hosts 主机名映射

添加主机名映射。使用与 `docker client --add-host` 参数相同的值。

```yaml
extra_hosts:
 - "somehost:162.242.195.82"
 - "otherhost:50.31.209.229"
```

则会在内部容器中/etc/hosts创建具有ip地址和主机名的主机名映射，例如：

```shell
162.242.195.82  somehost
50.31.209.229   otherhost
```

## healthcheck 健康检查

该HEALTHCHECK指令有两种形式：

- HEALTHCHECK [OPTIONS] CMD command （通过在容器内运行命令来检查容器运行状况）
- HEALTHCHECK NONE （禁用从基础镜像继承的任何运行状况检查）
- 
该HEALTHCHECK指令告诉Docker如何测试容器以检查它是否仍在工作。即使服务器进程仍在运行，也可以检测到陷入无限循环且无法处理新连接的Web服务器的健康情况。

Dockerfile中只能有一条指令。如果列出多个，则只有最后一个HEALTHCHECK生效;

命令的退出状态指示容器的运行状况。可能的值是：

0：成功 - 容器健康且随时可用
1：不健康 - 容器无法正常工作
2：保留 - 不要使用此退出代码

当容器指定了运行状况检查时，除了正常状态外，它还具有运行状况。这个状态最初是`starting`。当健康检查通过时，它就会变成`healthy`。经过一定数量的连续失败后，它就变成了`unhealthy`。

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

`test` 可以是 `NONE` 禁用任何检查； `CMD` 或者 `SHELL `

`interval` 检查间隔时间；

`timeout` 健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；

`retries` 当连续失败指定次数后，则将容器状态视为 `unhealthy` ，默认3次；

`start_period` 为需要时间引导的容器提供初始化时间。在此期间探测失败将不计入最大重试次数。但是，如果在启动期间运行状况检查成功，则会将容器视为已启动，并且所有连续失败将计入最大重试次数。


## init 
 没太懂  [官方文档-init](https://docs.docker.com/compose/compose-file/#init)

## isolation 

指定容器的隔离技术。在Linux上，唯一支持的值是default。在Windows中，可接受的值是default，process和 hyperv。

## labels 容器标签

docker-compose.yaml中三个labels的区别和具体作用：(建议使用反向DNS表示法)

build.labels 是设置镜像标签

deploy.labels 是设置服务service标签

labels 是设置容器标签

## logging 日志配置

这里单独写一篇文章学习。[官方文档-logging-1](https://docs.docker.com/compose/compose-file/#logging)  [官方文档-logging-2](https://docs.docker.com/config/containers/logging/configure/)

## network_mode 网络模式

```yaml
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

## networks 服务加入的网络

```yaml
services:
  some-service:
    networks:
     - some-network
     - other-network
```

## aliases 网络别名

相同的服务在不同的服务下可以又不同的别名

```yaml
version: "3.7"

services:
  web:
    image: "nginx:alpine"
    networks:
      - new

  worker:
    image: "my-worker-image:latest"
    networks:
      - legacy

  db:
    image: mysql
    networks:
      new:
        aliases:
          - database
      legacy:
        aliases:
          - mysql

networks:
  new:
  legacy:
```

## IPV4_ADDRESS，IPV6_ADDRESS  容器的静态IP地址

顶级 networks 中的相应网络配置，必须具有ipam包含每个静态地址的子网配置

```yaml
version: "3.7"

services:
  app:
    image: nginx:alpine
    networks:
      app_net:
        ipv4_address: 172.16.238.10
        ipv6_address: 2001:3984:3989::10

networks:
  app_net:
    ipam:
      driver: default
      config:
        - subnet: "172.16.238.0/24"
        - subnet: "2001:3984:3989::/64"
```

## PID 

跟主机系统共享进程命名空间。打开该选项的容器之间，以及容器和宿主机系统之间可以通过进程 ID 来相互访问和操作。

```yaml
pid: "host"
```

## ports 端口映射

以 HOST:CONTAINER 格式映射端口时，使用低于 60 的容器端口时可能会遇到错误的结果，因为YAML会将格式 xx:yy 中的数字解析为base-60值。因此，我们建议始终将端口映射明确指定为字符串。

```yaml
ports:
 - "3000"
 - "3000-3005"
 - "8000:8000"
 - "9090-9091:8080-8081"
 - "49100:22"
 - "127.0.0.1:8001:8001"
 - "127.0.0.1:5000-5010:5000-5010"
 - "6060:6060/udp"
```

或者

```yaml
ports:
  - target: 80
    published: 8080
    protocol: tcp
    mode: host
```

## restart 重启策略
 
 **使用（版本3）Compose文件在群集模式下部署堆栈时，将忽略此选项 。请改用   `deploy.restart_policy`**

## secrets 密钥

```yaml
version: "3.7"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    secrets:
      - my_secret
      - my_other_secret
secrets:
  my_secret:
    file: ./my_secret.txt
  my_other_secret:
    external: true
```
或者

```yaml
version: "3.7"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    secrets:
      - source: my_secret
        target: redis_secret
        uid: '103'
        gid: '103'
        mode: 0440
secrets:
  my_secret:
    file: ./my_secret.txt
  my_other_secret:
    external: true
```

## security_opt 标签覆盖

```yaml
security_opt:
  - label:user:USER
  - label:role:ROLE
```

## stop_grace_period SIGKILL等待时间

指定 stop_signal 在发送 SIGKILL 之前，如果它未处理 SIGTERM 信号，则尝试停止容器时要等待多长时间 。

默认情况下，stop在发送SIGKILL之前等待容器退出10秒。

```yaml
stop_grace_period: 1s
stop_grace_period: 1m30s
```

## stop_signal 停止信号

设置替代信号以停止容器。默认情况下stop使用 SIGTERM

```yaml
stop_signal: SIGUSR1
```

## sysctls 容器内核参数

```yaml
sysctls:
  net.core.somaxconn: 1024
  net.ipv4.tcp_syncookies: 0

或者

sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

## tmpfs 临时文件系统

在容器内安装临时文件系统。默认无限制。

## ulimits 容器的 ulimits 限制值。

例如，指定最大进程数为 65535，指定文件句柄数为 20000（软限制，应用可以随时修改，不能超过硬限制） 和 40000（系统硬限制，只能 root 用户提高）。

```yaml
ulimits:
  nproc: 65535
  nofile:
    soft: 20000
    hard: 40000
```

## volumes 

```yaml
version: "3.7"
services:
  web:
    image: nginx:alpine
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

  db:
    image: postgres:latest
    volumes:
      - "/var/run/postgres/postgres.sock:/var/run/postgres/postgres.sock"
      - "dbdata:/var/lib/postgresql/data"

volumes:
  mydata:
  dbdata:
```
`type`：安装类型volume，bind或tmpfs
`source`：mount的源，主机上用于绑定装载的路径，或顶级volumes键中定义的卷的名称 。不适用于tmpfs安装。
`target`：容器中安装卷的路径
`read_only`：flag将卷设置为只读
`bind`：配置其他绑定选项
`propagation`：用于绑定的传播模式
`volume`：配置其他卷选项
`nocopy`：flag用于在创建卷时禁用从容器复制数据
`tmpfs`：配置其他tmpfs选项
`size`：tmpfs mount的大小（以字节为单位）
`consistency`：mount的一致性要求，consistent，cached 或 delegated

---

参考

[官方文档-composefile](https://docs.docker.com/compose/compose-file/)

[Docker从入门到实践](https://yeasy.gitbooks.io/docker_practice/content/compose/compose_file.html)