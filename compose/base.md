# 安装及基本使用

## compose简介

**Compose** 项目是docker官方的开源项目，由python编写。负责实现对docker容器集群的快速编排。[docker-compose项目官方开源地址](https://github.com/docker/compose)

`Dockerfile` 模板，可以让用户定义一个单独的容器，而 `compose` 则可以来定义一组关联的应用容器为一个项目。比如一个web应用，使用nginx作为web服务端，php解析动态文件，mysql存储数据。三个服务可以在一个`docker-compose.yaml`文件中管理。

## 安装和卸载
Linux 系统：

下载Docker Comepose二进制文件

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`
```

添加可执行权限

```shell
chmod +x /usr/local/bin/docker-compose
```

可以参考官方文档安装，官方文档永远是最好的老师  [官方文档地址](https://docs.docker.com/compose/install/)

## 基本使用

### 创建项目目录，定义应用程序

```shell
# mkdir pyweb
# cd pyweb/
# vim app.py
```

### app.py 代码
```python
import time

import redis
from flask import Flask


app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)


def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)


@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

## 创建requirements.txt
```
flask
reids
```

## 创建 dockerfile
```yaml
FROM python:3.5-alpine
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

注意：官方使用得是 python:3.4-alpine 作为基础镜像，pip install会报错，改为3.5即可。

## 创建 docker-compose.file
```yaml
version: '3'
services:
  web:
    build: .
    ports:
     - "5000:5000"
  redis:
    image: "redis:alpine"
```

### 启动服务 `docker-compose up`

输出如下：
```
# docker-compose up
Building web
Step 1/5 : FROM python:3.5-alpine
3.5-alpine: Pulling from library/python
bdf0201b3a05: Pull complete
59c926705abf: Pull complete
a309f6bd7e2d: Pull complete
42d4929c1c07: Pull complete
fb82abe311e8: Pull complete
Digest: sha256:d17e05482015d2266b8135f74eef7d154b183a2a5dff511d42d253d9178056e6
Status: Downloaded newer image for python:3.5-alpine
 ---> 20aab83bbf15
Step 2/5 : ADD . /code
 ---> 57b0badbd270
Step 3/5 : WORKDIR /code
Removing intermediate container f268fbaf9c20
 ---> 908da9c70624
Step 4/5 : RUN pip install -r requirements.txt
 ---> Running in 1112957973f8
Collecting flask (from -r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/7f/e7/08578774ed4536d3242b14dacb4696386634607af824ea997202cd0edb4b/Flask-1.0.2-py2.py3-none-any.whl (91kB)
Collecting redis (from -r requirements.txt (line 2))
  Downloading https://files.pythonhosted.org/packages/ac/a7/cff10cc5f1180834a3ed564d148fb4329c989cbb1f2e196fc9a10fa07072/redis-3.2.1-py2.py3-none-any.whl (65kB)
Collecting click>=5.1 (from flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/fa/37/45185cb5abbc30d7257104c434fe0b07e5a195a6847506c074527aa599ec/Click-7.0-py2.py3-none-any.whl (81kB)
Collecting Werkzeug>=0.14 (from flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/18/79/84f02539cc181cdbf5ff5a41b9f52cae870b6f632767e43ba6ac70132e92/Werkzeug-0.15.2-py2.py3-none-any.whl (328kB)
Collecting itsdangerous>=0.24 (from flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/76/ae/44b03b253d6fade317f32c24d100b3b35c2239807046a4c953c7b89fa49e/itsdangerous-1.1.0-py2.py3-none-any.whl
Collecting Jinja2>=2.10 (from flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/1d/e7/fd8b501e7a6dfe492a433deb7b9d833d39ca74916fa8bc63dd1a4947a671/Jinja2-2.10.1-py2.py3-none-any.whl (124kB)
Collecting MarkupSafe>=0.23 (from Jinja2>=2.10->flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/b9/2e/64db92e53b86efccfaea71321f597fa2e1b2bd3853d8ce658568f7a13094/MarkupSafe-1.1.1.tar.gz
Building wheels for collected packages: MarkupSafe
  Building wheel for MarkupSafe (setup.py): started
  Building wheel for MarkupSafe (setup.py): finished with status 'done'
  Stored in directory: /root/.cache/pip/wheels/f2/aa/04/0edf07a1b8a5f5f1aed7580fffb69ce8972edc16a505916a77
Successfully built MarkupSafe
Installing collected packages: click, Werkzeug, itsdangerous, MarkupSafe, Jinja2, flask, redis
Successfully installed Jinja2-2.10.1 MarkupSafe-1.1.1 Werkzeug-0.15.2 click-7.0 flask-1.0.2 itsdangerous-1.1.0 redis-3.2.1
Removing intermediate container 1112957973f8
 ---> 3093fe39465f
Step 5/5 : CMD ["python", "app.py"]
 ---> Running in 35972a01e7df
Removing intermediate container 35972a01e7df
 ---> bb9dfca400d5
Successfully built bb9dfca400d5
Successfully tagged pyweb_web:latest
WARNING: Image for service web was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Pulling redis (redis:alpine)...
alpine: Pulling from library/redis
bdf0201b3a05: Already exists
542e0c4f2f18: Pull complete
cbf113c39f65: Pull complete
09158274ea6c: Pull complete
ffc2a2e9a3a6: Pull complete
bcdc222d2d8e: Pull complete
Digest: sha256:ef67270b8bcb6b801a67d186ed16051520867b985e4f2f1233b510938dc55b22
Status: Downloaded newer image for redis:alpine
Creating pyweb_redis_1 ... done
Creating pyweb_web_1   ... done
Attaching to pyweb_redis_1, pyweb_web_1
redis_1  | 1:C 22 Apr 2019 08:43:29.101 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis_1  | 1:C 22 Apr 2019 08:43:29.101 # Redis version=5.0.4, bits=64, commit=00000000, modified=0, pid=1, just started
redis_1  | 1:C 22 Apr 2019 08:43:29.101 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
redis_1  | 1:M 22 Apr 2019 08:43:29.103 * Running mode=standalone, port=6379.
redis_1  | 1:M 22 Apr 2019 08:43:29.103 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
redis_1  | 1:M 22 Apr 2019 08:43:29.103 # Server initialized
redis_1  | 1:M 22 Apr 2019 08:43:29.103 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
redis_1  | 1:M 22 Apr 2019 08:43:29.103 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
redis_1  | 1:M 22 Apr 2019 08:43:29.103 * Ready to accept connections
```