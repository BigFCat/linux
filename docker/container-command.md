# 容器contain操作命令

## 启动容器
1. 执行命令后即退出容器
```shell
# docker run busybox /bin/echo 'hello world'
hello world
# 
```
2. 交互式，启动容器后进入容器终端，`exit`退出容器，容器停止
```shell
# docker run -it busybox /bin/sh
/ # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
/ # hostname 
19e828cb7a3c
/ # exit
# 
```
3. 后台运行
```shell
# docker run -d busybox /bin/sh -c "while true; do echo hello world; sleep 1; done"
f26e433c8dfe6b94a6200d0fe769aa620d0446abe16f11d34415a6c53893b5ab
# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
f26e433c8dfe        busybox             "/bin/sh -c 'while t…"   3 seconds ago       Up 2 seconds                            peaceful_brattain
```

以上： `-t` 选项让docker分配一个伪终端，并绑定到容器得标准输入上；`-i` 则让容器得标准输入保持打开；`-d` 后台运行。如果不使用`-d`参数，容器会将输出得结果打印在宿主机上；如果使用了该参数，则不会打印到宿主机上，可以使用 `docker logs` 命令查看容器日志。另外，容器能否长久运行，和容器的运行前台进程有关，和`-d`无关。

使用 `docker run` 命令创建容器时，docker在后台运行的标准操作流程：

1. 检查本地是否存在指定的镜像，不存在就从公有仓库下载
2. 利用镜像创建并启动一个容器
3. 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
4. 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
5. 从地址池配置一个 ip 地址给容器
6. 执行用户指定的应用程序
7. 执行完毕后容器被终止

## 终止

`docker stop`

## 进入容器

使用`docker exec -it ` 即可进入正在运行的容器；`docker attach` 进入容器后`exit`退出会导致容器停止。

## 导入和导出

`docker export`  

`docker import`

## 删除

`docker rm`  

如果删除运行中的容器，可以添加`-f` 参数，docker会发送`SIGKILL`信号给容器。