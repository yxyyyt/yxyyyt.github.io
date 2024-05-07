# 常用参数

- `-d` 后台运行容器并打印容器ID
- `--name` 为容器指定一个名称
- `-i` 保持STDIN打开，即使未连接
- `-t` 分配一个伪TTY
- `-a` 连接STDOUT/STDERR并转发信号
- `--link` 可以用来链接2个容器，使得源容器（被链接的容器）和接收容器（主动去链接的容器）之间可以互相通信，并且接收容器可以获取源容器的一些数据，如源容器的环境变量。
- `-d` 指定网络驱动
- `-p` 端口映射，自动暴露
- `--expose` 暴露端口，但未做映射
- `-P` 会自动映射Dockerfile中 EXPOSE xxx 声明的端口到主机的任意端口
- `-v` 挂载卷
- `--restart always` 在重启docker时，自动启动容器



# 启动

```shell
systemctl start docker
```



# 命令

## docker attach 🔖

连接本地标准输入，输出和错误流到<font color=red>正在运行</font>的container。

- 以 `-i -t` 运行，<font color=red>使用 `CTRL+p+q` 与container分离</font>。
- `CTRL-c` 停止container

```shell
# -d Run container in background and print container ID (--detach)
# --name Assign a name to the container
# -i Keep STDIN open even if not attached
# -t Allocate a pseudo-TTY
docker run --name test -d -it debian
docker attach test
```



## docker build

从Dockerfile和上下文构建image。

上下文是指定 `PATH or URL` 参数的文件集合。构建进程可以引用上下文中的文件。可以指定一个Git仓库作为其**URL**，在本机首先拉取仓库到一个临时目录，成功后发送到Docker daemon作为其上下文。也可以指定本地文件系统的一个目录作为**PATH**。

- <font color=red>构建命令默认会在构建上下文的root下寻找Dockerfile，可以使用 `-f or --file` 指定代替Dockerfile，适用于一个目录下存在多个Dockerfile进行不同的构建</font>

```shell
# git repository # tag or branch（指定分支或tag，默认master） : /docker（指定上下文，默认/）
docker build https://github.com/docker/rootfs.git#container:docker

# PATH是.作为构建上下文
docker build .

# 下载tar.gz，ctx/Dockerfile是其内部Dockerfile的位置
# -f 指定Dockerfile，后面参数指定上下文
docker build -f ctx/Dockerfile http://server/ctx.tar.gz

# 从标准输入读入一个Dockerfile，没有上下文
docker build - < Dockerfile

# 从标准输入读入一个tar.gz
docker build - < context.tar.gz

# 2.0 为image打tag
# -t: 镜像的名字及标签，通常 name:tag 或者 name 格式；可以在一次构建中为一个镜像设置多个标签
docker build -t vieux/apache:2.0 .
```



## docker commit

以改变后的容器为基，创建一个新的image。

```shell
# c3f279d17e0a 容器id
# svendowideit/testimage REPOSITORY
# version3 TAG
docker commit c3f279d17e0a  svendowideit/testimage:version3
```



## docker container attach

连接本地标准输入，输出和错误流到<font color=red>正在运行</font>的container。



## docker container commit

以改变后的容器为基，创建一个新的image。



## docker container cp

在容器和本地文件系统之间拷贝文件或文件夹。



## docker container create

创建一个新的容器。



## docker container exec

在一个运行的container内执行一个命令。



## docker container inspect

显示容器的详细信息。



## docker container kill

杀死一个或多个正在运行的container。



## docker container ls 🔖

列举container。



## docker container pause

暂停一个或多个container的所有进程。



## docker container port

列举容器的所有端口映射或一个指定映射。



## docker container rename

重命名一个容器。



## docker container restart

重启一个或多个容器。



## docker container rm

移除一个或多个容器。



## docker container run

在一个新的容器运行一个命令。



## docker container start

启动一个或多个停止的容器。



## docker container stats

显示容器实时资源使用情况统计信息。



## docker container stop

停止一个或多个正在运行的容器。



## docker container top

显示一个容器正在运行的进程。



## docker container unpause

取消暂停一个或多个container的所有进程。



## docker cp 🔖

在容器和本地文件系统之间拷贝文件或文件夹。

```shell
# 本地目录 -> 容器的/www/目录下
docker cp /www/runoob 96f7f14e99ab:/www/
# 本地目录 -> 容器的/目录下，改名为www
docker cp /www/runoob 96f7f14e99ab:/www
# 容器的/www目录 -> 本地目录
docker cp  96f7f14e99ab:/www /tmp/
```



## docker create 🔖

创建一个新的容器。

```shell
# 创建未运行容器
docker create -t -i fedora bash
# 启动
# -a attach
docker start -a -i 6d8af538ec5

# -v mount a volume
# --name Assign a name to the container
docker create -v /data --name data ubuntu
# --rm Automatically remove the container when it exits
# --volumes-from Mount volumes from the specified container(s)
docker run --rm --volumes-from data ubuntu ls -la /data

# -v 本地目录:容器卷
docker create -v /home/docker:/docker --name docker ubuntu
docker run --rm --volumes-from docker ubuntu ls -la /docker
```



## docker exec 🔖

在一个运行的container内执行一个命令。在容器内的默认目录，也可以通过Dockerfile的 `WORKDIR` 指令指定。

```shell
# 创建一个名称为ubuntu_bash的容器，并启动一个Bash会话
docker run --name ubuntu_bash --rm -i -t ubuntu bash
# 创建一个/tmp/execWorks文件在运行的ubuntu_bash容器
docker exec -d ubuntu_bash touch /tmp/execWorks
```



## docker image build

从Dockerfile和上下文构建image。



## docker image inspect

显示镜像的详细信息。



## docker image ls

列举image。



## docker image pull

从一个注册中心拉取一个image或一个repository 。



## docker image push

向一个注册中心推送一个image或一个repository 。



## docker image rm

删除一个或多个image。



## docker info

显示docker系统范围的信息。

```shell
docker info
```



## docker inspect

显示docker底层信息。

```shell
# 显示image详细配置信息
docker inspect imageid
```



## docker kill

杀死一个或多个正在运行的container。

```shell
docker kill my_container
```



## docker logs 🔖

获取容器日志

```shell
#  -f 继续从容器的 STDOUT 和 STDERR 持续输出流日志
docker logs -f my_container
```



## docker network connect

将**正在运行的容器**连接到网络。一旦连接，在同一个网络中容器可以同其他容器通讯。

```shell
# 正在运行的容器连接到网络
docker network connect multi-host-network container1

# --network 启动容器并立即连接到网络
docker run -itd --network=multi-host-network busybox

# --ip 指定ip
docker network connect --ip 10.10.36.122 multi-host-network container2

# --link 连接到其他容器，c1指定的别名
docker network connect --link container1:c1 multi-host-network container2
```



## docker network create

创建一个网络。内建的网络驱动是 `bridge or overlay` ，如果不指定 `--driver` ，命令自动创建一个 `bridge` 网络。当安装Docker Engine后，系统自动创建一个 `docker0` bridge网络。当使用命令 `docker run` 启动一个新的容器，自动连接到bridge网络。此默认bridge网络不可以删除。

- `bridge` 网络在单个Engine上隔离网络。如果建立一个跨越多个Engine，必须创建 `overlay` 网络。

```shell
# -d driver
docker network create -d bridge my-bridge-network

# 172.28.0.0/16 前16位固定 172.28.0.0~172.28.255.255
# 172.28.5.0/24 前24位固定 172.28.5.0~172.28.5.255
docker network create \
  --driver=bridge \
  --subnet=172.28.0.0/16 \
  --ip-range=172.28.5.0/24 \
  --gateway=172.28.5.254 \
  br0
```



## docker network disconnect

断开容器与网络的连接。

```shell
docker network disconnect multi-host-network container1
```



## docker network inspect

显示网络的详细信息。

```shell
# 默认三个driver，name分别是bridge, host, none

# -o "com.docker.network.bridge.enable_icc"=true 启用或禁用容器间连接
# -o "com.docker.network.bridge.name"=docker0 虚拟网卡docker0
docker network inspect bridge
```



## docker network ls

列举network

```shell
docker network ls
```



## docker network rm

移除一个或多个network

```shell
docker network rm my-network
```



## docker pause

暂停一个或多个container的所有进程。

```shell
docker pause my_container
```



## docker port 🔖

列举容器的所有端口映射或一个指定映射。

```shell
# 7890/tcp -> 0.0.0.0:4321 容器端口 -> 本机端口
docker port test
```



## docker ps

列举容器。

```shell
# -a 显示所有容器。默认只显示正在运行的容器
docker ps -a
```



## docker pull

从一个注册中心拉取一个image或一个repository 。

```shell
# 默认拉取 latest
docker pull mysql:5.7.32
```



## docker push

向一个注册中心推送一个image或一个repository 。

```shell
# 创建image
docker container commit c16378f943fe rhel-httpd:latest
# 创建tag
# docker image tag source target
# 1. 为镜像创建标签，创建的新标签指向的是原镜像。
# 2. 在Docker Hub共享镜像，必须命名为<Docker Hub ID>/<Repository Name>:<tag>的样式。
docker image tag rhel-httpd:latest registry-host:5000/myadmin/rhel-httpd:latest
# 上传注册中心
docker image push registry-host:5000/myadmin/rhel-httpd:latest
```



## docker rename

重命名一个容器。

```shell
docker rename my_container my_new_container
```



## docker restart

重启一个或多个容器。

```shell
docker restart my_container
```



## docker rm 🔖

移除一个或多个容器。

```shell
docker rm /redis
```



## docker rmi 🔖

删除一个或多个image。

```shell
# 删除与id匹配的所有image
docker rmi -f fd484f19954f
```



## docker run 🔖

在一个新的容器运行命令。

- 如果本地没有ubuntu镜像，则从配置中心下载，相当于 `docker pull ubuntu`
- 创建一个新的容器，相当于 `docker container create`
- 分配一个可读写的文件系统给容器作为最后一层
- 创建一个网络接口，将容器连接到默认网络，为容器分配IP地址
- 启动容器，执行 `/bin/bash`
- 输入exit命令，容器stop。可以重新start或者remove容器

`-p -P --expose` 区别

- -p 端口映射，自动暴露
- --expose 暴露端口，但未做映射
- -P 会自动映射Dockerfile中 EXPOSE xxx 声明的端口到主机的任意端口

```shell
docker run --name test -it debian

# -w 设置工作目录
docker run -w /path/to/dir/ -i -t  ubuntu pwd

# -v 映射本地目录->容器目录
docker run -v `pwd`:`pwd` -w `pwd` -i -t ubuntu pwd

# 绑定本机的80端口到容器的8080端口
# ip:hostPort:containerPort，ip和hostPort可省略
# 发布的端口默认暴露
docker run -p 127.0.0.1:80:8080/tcp ubuntu bash

# --expose 暴露端口，但未做映射；可以结合--net=host的方式，使得容器直接占用主机端口，从而访问宿主机端口就可以直接访问容器端口提供的服务
docker run --expose 80 ubuntu bash

# 后台运行，返回容器id、
docker run -itd --name mysqlnode mysql:5.7.32
```



## docker search

在Docker Hub中查找镜像。

```shell
docker search busybox
```



## docker start 🔖

启动一个或多个停止的容器。

```shell
# -a Attach STDOUT/STDERR and forward signals
# -i Attach container’s STDIN
docker start my_container
```



## docker stop 🔖

停止一个或多个正在运行的容器。

```shell
docker stop my_container
```



## docker tag

为源镜像创建目标镜像的tag。

```shell
# SOURCE_IMAGE[:TAG] -> TARGET_IMAGE[:TAG]
docker tag 0e5574283393 fedora/httpd:version1.0
```



## docker top

显示一个容器正在运行的进程。



## docker unpause

取消暂停一个或多个container的所有进程。

```shell
docker unpause my_container
```



## docker version

显示docker的版本信息。

```shell
docker version
```



## docker volume create

创建卷。

```shell
docker volume create hello
# -v 挂载hello卷
# -d --driver 默认local
docker run -d -v hello:/world busybox ls /world
```



## docker volume inspect

显示卷的详细信息。

```shell
docker volume inspect myvolume
```



## docker volume ls

列举卷。

```shell
docker volume ls
```



## docker volume prune

删除所有未使用的local卷。

```shell
docker volume prune
```



## docker volume rm

删除一个或多个卷。

```shell
docker volume rm hello
```

