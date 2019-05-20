# Swarm 配置管理及密钥管理

在这篇笔记之前，在一个服务中，配置文件应用到容器中都是写在 dockerfile 的 COPY 命令中一起打包到镜像中；又或者是单独的挂载到容器中，当然使用nfs作为文件共享也没什么问题。但是有了官方的服务配置解决方案为啥不用呢？

Docker 17.06版本后引入了swarm服务配置，可以将非敏感数据，如配置文件存储在容器之外，无需将配置文件绑定到容器或者使用环境变量，甚至可以将配置与环境变量或者标签结合使用，以获得最大的灵活性。

## Docker是如何管理这些配置的？
创建添加一个配置到swarm集群后，config 会保存在raft日志中，(raft是manager节点之间通信保持数据一致性的协议？不知道这么说对不对)。raft_log会同步到各个manager节点，保证整个集群的数据的一致性。

新创建一个服务访问配置信息时，**默认将文件挂载在`/<my-config-name>`目录下**

### 创建一个config配置文件
创建config必须先要有配置文件存在，以nginx为示例学习

```shell
# cat nginx.conf 
server {
    listen  81;
    server_name  _;
    location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
    }
}

[root@node1 config]# docker config create MyNginxConf nginx.conf 
q4ea2zlx7o323o12dp4uqwms3

[root@node1 config]# docker config ls
ID                          NAME                CREATED             UPDATED
q4ea2zlx7o323o12dp4uqwms3   MyNginxConf         4 seconds ago       4 seconds ago

[root@node1 config]# docker config inspect n99
[
    {
        "ID": "n994fum9of6n9mt3f7z9mkmfm",
        "Version": {
            "Index": 3212
        },
        "CreatedAt": "2019-04-26T08:19:43.269137253Z",
        "UpdatedAt": "2019-04-26T08:19:43.269137253Z",
        "Spec": {
            "Name": "MyNginxConf1.0",
            "Labels": {},
            "Data": "c2VydmVyIHsKICAgIGxpc3RlbiAgODE7CiAgICBzZXJ2ZXJfbmFtZSAgXzsKICAgIGxvY2F0aW9uIC8gewogICAgICAgICAgICByb290IC91c3Ivc2hhcmUvbmdpbngvaHRtbDsKICAgICAgICAgICAgaW5kZXggaW5kZXguaHRtbCBpbmRleC5odG07CiAgICB9Cn0K"
        }
    }
]

```

配置内容通过加密的形式存储，不过这个加密很容易解密！base64解密。

### 创建service服务，并使用config
```shell
docker service create --name my-web-nginx --config src=MyNginxConf,target=/etc/nginx/conf.d/test.conf --publish 8081:81 nginx
```

### 更新config
更新config，需要先创建一个新的名称不重复的config。然后升级服务，删除容器中旧的config并挂载新的配置。

```shell
# docker config ls
ID                          NAME                CREATED             UPDATED
q4ea2zlx7o323o12dp4uqwms3   MyNginxConf         28 minutes ago      28 minutes ago
n994fum9of6n9mt3f7z9mkmfm   MyNginxConf1.0      8 minutes ago       8 minutes ago
```

```shell
docker service update --config-rm MyNginxConf --config-add source=MyNginxConf1.0,target=/etc/nginx/conf.d/test.conf my-web-nginx
```
## 密钥管理

### 创建secret
```shell
# openssl rand -base64 20 | docker secret create mysql_root_password -
jhbinu21rqkvhauq2o9n7lorq
[root@node1 config]# docker secret ls
ID                          NAME                  DRIVER              CREATED              UPDATED
rse0l0ybkvobhjtg4o12bmh9n   mysql_password                            About a minute ago   About a minute ago
jhbinu21rqkvhauq2o9n7lorq   mysql_root_password                       6 seconds ago        6 seconds ago
[root@node1 config]# docker secret inspect rs
[
    {
        "ID": "rse0l0ybkvobhjtg4o12bmh9n",
        "Version": {
            "Index": 3225
        },
        "CreatedAt": "2019-04-26T09:25:11.892257753Z",
        "UpdatedAt": "2019-04-26T09:25:11.892257753Z",
        "Spec": {
            "Name": "mysql_password",
            "Labels": {}
        }
    }
]
```

**在管理配置信息config中，创建的数据虽然经过加密，但是任然能解密，不一定安全。secret则没用保存相关数据**

### 创建mysql服务

```shell
docker network create -d overlay mysql_private

docker service create \
     --name mysql \
     --replicas 1 \
     --network mysql_private \
     --mount type=volume,source=mydata,destination=/var/lib/mysql \
     --secret source=mysql_root_password,target=mysql_root_password \
     --secret source=mysql_password,target=mysql_password \
     -e MYSQL_ROOT_PASSWORD_FILE="/run/secrets/mysql_root_password" \
     -e MYSQL_PASSWORD_FILE="/run/secrets/mysql_password" \
     -e MYSQL_USER="wordpress" \
     -e MYSQL_DATABASE="wordpress" \
     mysql:latest

```

如果你没有在 target 中显式的指定路径时，secret 默认通过 tmpfs 文件系统挂载到容器的 /run/secrets 目录中。

### 创建wordpress服务
```shell
docker service create \
     --name wordpress \
     --replicas 1 \
     --network mysql_private \
     --publish published=30000,target=80 \
     --mount type=volume,source=wpdata,destination=/var/www/html \
     --secret source=mysql_password,target=wp_db_password,mode=0400 \
     -e WORDPRESS_DB_USER="wordpress" \
     -e WORDPRESS_DB_PASSWORD_FILE="/run/secrets/wp_db_password" \
     -e WORDPRESS_DB_HOST="mysql:3306" \
     -e WORDPRESS_DB_NAME="wordpress" \
     wordpress:latest
```