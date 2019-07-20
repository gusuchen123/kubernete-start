# 一、Docker 基础使用

## 1、Docker安装和卸载

- 更新yum源

  ```bash
  $ yum install -y wget
  $ mkdir -p /etc/yum.repos.d/bak
  $ mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak/
  ## 更新国内yum源
  $ wget -O /etc/yum.repos.d/Centos-Base.repo \
    > http://mirrors.cloud.tencent.com/repo/centos7_base.repo
  
  ## 更新国内yum源
  $ wget -O /etc/yum.repos.d/epel.repo \
    > http://mirrors.cloud.tencent.com/repo/epel-7.repo
  $ yum clean all && yum makecache
  $ yum install -y git vim gcc glibc-static telnet bridge-utils net-tools
  ```

- 添加docker.repo（使用阿里云的yum源，可以提升速度）

  ```bash
  wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo \
  -O /etc/yum.repos.d/docker-ce.repo
  ```

- 添加 kubernetes.repo

  ```bash
  $ tee /etc/yum.repos.d/kubernetes.repo <<-'EOF'
  [kubernetes]
  name=Kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
  EOF
  $ yum clean all && yum makecache
  ```

  

- 安装 docker

  参考官方文档：https://docs.docker.com/install/linux/docker-ce/centos/

  ```bash
  $ sudo rpm -qa | grep docker
  ## 删除已安装的docker软件
  $ sudo yum remove docker \
                    docker-client \
                    docker-client-latest \
                    docker-common \
                    docker-latest \
                    docker-latest-logrotate \
                    docker-logrotate \
                    docker-engine
  $ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
  ## 安装docker
  $ sudo yum install -y docker-ce docker-ce-cli containerd.io
  ## 设置开机自启动
  $ sudo systemctl enable docker
  ## 启动docker服务
  $ sudo systemctl start docker
  ## 验证docker
  $ sudo docker run hello-world
  $ docker version
  Client:
   Version:           18.09.7
   API version:       1.39
   Go version:        go1.10.8
   Git commit:        2d0083d
   Built:             Thu Jun 27 17:56:06 2019
   OS/Arch:           linux/amd64
   Experimental:      false
  
  Server: Docker Engine - Community
   Engine:
    Version:          18.09.7
    API version:      1.39 (minimum version 1.12)
    Go version:       go1.10.8
    Git commit:       2d0083d
    Built:            Thu Jun 27 17:26:28 2019
    OS/Arch:          linux/amd64
    Experimental:     false
  ```

- 如果网速很快可以尝试下载安装脚本进行安装

  ```bash
  $ tee setup.sh <<-'EOF'
  #/bin/sh
  
  # install some tools
  sudo yum install -y git vim gcc glibc-static telnet bridge-utils
  
  # install docker
  curl -fsSL get.docker.com -o get-docker.sh
  sh get-docker.sh
  
  # start docker service
  sudo groupadd docker
  sudo gpasswd -a vagrant docker
  sudo systemctl start docker
  
  rm -rf get-docker.sh
  EOF
  ```

  

- 配置阿里云docker加速器

  ```bash
  sudo mkdir -p /etc/docker
  sudo tee /etc/docker/daemon.json <<-'EOF'
  {
    "registry-mirrors": ["https://741gqwq9.mirror.aliyuncs.com"]
  }
  EOF
  sudo systemctl daemon-reload
  sudo systemctl restart docker
  ```

- 启动服务

  ```bash
  #设置 docker 开机服务启动
  $ systemctl enable docker.service 
  #立即启动 docker 服务
  $ systemctl daemon-reload && systemctl restart docker
  ```

- docker容器的配置文件

  ```bash
  /usr/lib/systemd/system/docker.service                --docker 配置文件地址
  ```

- docker-compose安装

  下载并安装docker-compose(官网推荐)

  ```bash
  $ curl -L https://github.com/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
  $ chmod +x /usr/local/bin/docker-compose
  ```

  由于下载缓慢，建议离线安装

  https://github.com/docker/compose/releases

  https://docs.docker.com/compose/install/

  ```bash
  ## 下载docker-compose-Linux-x86_64
  $ wget https://github.com/docker/compose/releases/download/1.24.0/docker-compose-Linux-x86_64
  $ cp docker-compose-Linux-x86_64 /usr/local/bin/
  $ cd /usr/local/bin/
  $ mv docker-compose-Linux-x86_64 docker-compose
  $ chmod +x docker-compose
  ## 验证是否成功
  $ docker-compose version
  docker-compose version 1.24.0, build 0aa59064
  docker-py version: 3.7.2
  CPython version: 3.6.8
  OpenSSL version: OpenSSL 1.1.0j  20 Nov 2018
  ```

## 2、Docker的3个概念

- image: 镜像
  - 类似于虚拟机中的镜像
  - 是一个包含文件系统的面向Docker引擎的**只读**模板
  - 任何应用程序运行都需要环境，而镜像就是用来提供这种运行环境的
- container: 容器
  - 类似于轻量级的沙盒，可以看作是一个极简的Linux系统环境(包括root权限、进程空间、用户空间和网络空间等)，以及运行在其中的应用程序。
  - Docker Engine 利用容器来运行、隔离各个应用。
  - 容器是镜像创建的应用实例，可以创建、启动、停止、删除容器，各个容器之间相互隔离，互不影响。
  - 镜像和容器的关系类比于：**实例和类**
  - 镜像本身是只读的，容器从镜像启动时，Docker在镜像的上层创建了一个可写层，镜像本身不发生变化。
- repository: 仓库
  - 类比于代码仓库，这里是镜像仓库，是Docker用来集中存放镜像文件的地方。
  - 注册服务器(Registry)：注册服务器是存放仓库的地方，一般会有多个仓库。
  - 仓库是存放镜像文件的地方，一般每个仓库存放一类镜像，每个镜像利用tag来区分。

## 3、Docker中关于镜像的基本操作

- 从官方注册服务器下载镜像

  ```bash
  ## 查看 docker 相关信息
  $ docker info
  ## 查看 docker 服务端和客户端相关版本
  $ docker version
  ## 查询docker镜像
  $ docker search '镜像名称'
  $ docker search centos
  ## 下载或更新 docker 镜像 <docker pull '镜像名称':'版本'>
  $ docker pull '镜像名称:版本号'
  $ docker pull centos
  ## 查看本地已经下载的镜像，docker image ls -a
  $ docker images
  ## 查看 docker image 所有的命令
  $ docker image --help
  ```

- 利用镜像启动一个容器后进性修改 ==> 利用docker container commit 提交更新后的副本

  `docker container commit: Create a new image from a container's changes`

  ```shell
  ## docker run:执行容器，-it:打开命令终端，java:执行名叫java的镜像
  ## java -version:打开命令终端后执行该命令
  $ docker run -it java:latest java -version
  ## 以命令行的方式进入 openjdk 镜像
  $ docker run -it --entrypoint bash openjdk:7-jre
  ## 查看名叫 java 的 docker 镜像的环境变量
  $ docker run java env
  ## 容器将会运行在后台模式。
  $ docker run 后面追加-d=true或者-d
  ## 容器运行完成后立即删除
  $ docker run 后面追加 -rm
  ## 进入到到该容器中，或者attach重新连接容器的会话 
  ## <不建议使用 attach，一个是会话的原因，一个是断开后会停止 docker 镜像>
  $ docker exec
  ## 启动一个容器
  $ docker run -it -rm centos:latest /bin/bash
  [root@72f1a8a0e394 /]# yum install -y git
  [root@72f1a8a0e394 /]# git --verison
  [root@72f1a8a0e394 /]# exit
  ## 查看所有的容器，docker container ls -a
  $ docker ps -a
  
  ## 将容器转化成一个新的镜像, -m:指定说明信息，-a:指定用户信息，72f1a8:容器ID
  ## gusuchen163/centos:git指定目标镜像的用户名、仓库名、tag信息
  $ docker commit -m "centos with git" -a "gusuchen" 72f1a8 gusuchen163/centos:git
  $ docker images
  ```

- 利用Dockerfile创建镜像（推荐操作）

  `docker image build: Build an image from Dockerfile`

  ```shell
  # 说明该镜像以哪一个镜像为基础，使用 base image
  FROM centos:latest
  
  # 构造者基本信息
  LABEL maintainer="gu_suchen@163.com"
  LABEL version="1.0"
  LABEL description="this is description"
  
  # 在build镜像时执行的操作
  RUN yum update -y && yum install -y vim \
      python-dev
  
  # 拷贝本地的文件到镜像中
  COPY ./* /usr/share/gitdir/
  ```

  利用docker build 构建镜像：

  ```shell
  $ docker build -t "gusuchen163/centos:gitdir" .
  $ docker images
  ```

  ```shell
  # 删除镜像：先删除容器才能删除镜像
  $ docker container rm container_name/container_id
  $ docker rm cotainer_name/container_id
  
  $ docker image rm image_name/image_id
  $ docker rmi image_name/image_id
  
  # 保存镜像到本地:-o等同于--output
  $ docker save -o centos.tar gusuchen163/centos:git
  
  # 加载本地镜像:-i等同于--input
  $ docker load -i centos.tar
  ```
  
  对image打标签
  
  ```bash
  ## 下载镜像
  $ docker pull zookeeper:3.5
  ## docker 登陆，然后提示 输入用户名，密码
  $ docker login
  ## 为镜像 zookeeper 打上tag <:3.5是镜像版本>
  $ docker tag zookeeper:3.5 test/zookeeper:3.5
  ## 将zookeeper 镜像上传到仓库<:3.5是镜像版本>
  $ docker push test/zookeeper:3.5
  ```

## 4、Docker中关于容器的基本操作

- 基于镜像启动容器

  ```shell
  ## 容器生命周期相关指令
  $ docker create/start/stop/pause/unpause
  ## 创建一个 docker 容器名叫 myjava，后面的命令就是容器所要做的事情
  $ docker create -it --name=myjava java java -version
  ## 启动刚刚创建的容器<测试>
  $ docker start myjava
  ## 建一个 mysql 的 docker 容器，-e是往容器环境变量里面设值<一般服务的配置也是通过这个传递的>，-p 3306:3306 是将主机的3306转发到容器的3306端口《可使用--net=host替代直接使用宿主机端口不做转发》，最后的mysql是镜像的名称
  $ docker create --name=mysql -e MYSQL_ROOT_PASSWORD=jiang -p 3306:3306 mysql
  ## 启动刚刚创建的 mysql docker容器
  $ docker start '容器名称 || 容器ID'
  ## 进入容器 <如果报没有 bash 的错误直接使用：docker exec -it '容器名称' sh>进入容器
  $ docker exec -it '容器名称 || 容器ID' bash
  ## 停止 mysql docker容器
  $ docker stop '容器名称'
  ## 查看 docker 所有的容器
  $ docker ps -a
  ## 查看 dicker 当前正在运行的容器
  $ docker ps
  $ docker container ls -a
  ## 删除容器  'id' 就是 docker ps -a 查看到的id
  $ docker rm 'id'
  ## 查看 服务容器运行日志 <就是用docker ps命令显示出来的那个容器ID>, -f:追踪的方式查看实时日志
  $ docker logs -f '容器ID' 
  ## 将容器创建为镜像 'id'=容器ID
  $ docker commit 'id' 镜像名称
  ## -it 以交互式的方式启动容器，exit退出，容器停止。
  ## 如果想让容器一直运行，而不是停止，可以使用快捷键 ctrl+p ctrl+q 退出，此时容器的状态为Up。
  $ docker run -it centos:latest /bin/bash 
  
  $ docker run centos:latest /bin/bash -c "while true;do echo hello;sleep 1;done"
  
  ## 使用了-d参数，使这个容器处于后台运行的状态，不会对当前终端产生任何输出
  ## 所有的stdout都输出到log，可以使用docker logs container_name/container_id查看。
  $ docker run -d centos:latest /bin/bash \
    > -c "while true;do echo hello;sleep 1;done"
  ## 查看容器内的日志
  $ docker logs container_name/container_id
  ```

- 容器启动、停止、重启

  ```shell
  $ docker start container_name/container_id
  $ docker stop container_name/container_id
  $ docker restart container_name/container_id
  ```

- 启动容器之后，进入这个容器里面：docker attach（不推荐）

  ```bash
  ## 不推荐
  $ docker attach container_name/container_id
  ## 推荐
  $ docker exec -it container_name/container_id /bin/bash
  ```

- 删除容器

  ```bash
  $ docker rm container_name/container_id
  $ docker rm $(docker container ls -aq)
  $ docker rmi $(docker image ls -aq)
  $ docker container ls -a | awk {'print$1'}
  $ docker container ls -f "status=exited" -q
  $ docker rm $(docker container ls -f "status=exited -q")
  ```

- docker exec: Run a command in a running containe

  ```shell
  $ docker exec -it 930f6d5fecff /bin/bash
  $ docker exec -it 930f6d5fecff python
  $ docker exec -it 930f6d5fecff ip addr
  ```

- 启动 | 停止 | 重启 | 删除

  ```shell 
  $ docker run -d --name=demo gusuchen163/flask-hello-world ## 后台启动，指定名字
  $ docker stop gusuchen163/flask-hello-world
  $ docker start gusuchen163/flask-hello-world
  $ docker restart gusuchen163/flask-hello-world
  ```

- docker inspect:  Return low-level information on Docker objects

  ```shell
  $ docker inspect 930f6d5fecff ## 查询容器的详细信息
  ```

- docker logs:   Fetch the logs of a container

  ```shell
  $ docker logs 930f6d5fecf ## 查询容器运行产生的输出日志
   * Serving Flask app "app" (lazy loading)
   * Environment: production
     WARNING: This is a development server. Do not use it in a production deployment.
     Use a production WSGI server instead.
   * Debug mode: off
   * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
  127.0.0.1 - - [14/Jun/2019 06:59:32] "GET / HTTP/1.1" 200 -
  ```

## 5、Docker中关于仓库的基本操作

Docker官方维护了一个DockerHub的公共仓库，里边包含有很多平时用的较多的镜像。除了从上边下载镜像之外，我们也可以将自己自定义的镜像发布（push）到DockerHub上。

#### 5-1.镜像的上传和下载

- 访问[https://hub.docker.com/](https://link.zhihu.com/?target=https%3A//hub.docker.com/)，如果没有账号，需要先注册一个。

- 利用命令docker login登录DockerHub，输入用户名、密码即可登录成功：

  ```shell
  $ docker login
  Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
  Username: gusuchen163
  Password:
  Login Succeeded
  ```

- 将本地的镜像推送到DockerHub上，这里的gusuchen163要和登录时的username一致：

  ```shell
  $ docker push gusuchen163/centos:git ## 成功推送
  $ docker push xxx/centos:git ## 失败
  The push refers to a repository [docker.io/xxx/centos]
  unauthorized: authentication required
  ```

- 以后别人就可以从你的仓库中下载合适的镜像了：

  ```shell
  $ docker pull gusuchen163/centos:git
  ```

#### 5-2、镜像的创建有两种方法，镜像的更新也有两种方法

- 创建容器之后做更改，之后commit生成镜像，然后push到仓库中。
- 更新Dockerfile。在工作时一般建议这种方式，更简洁明了。

#### 5-3、镜像、容器、仓库

从仓库（一般为DockerHub）下载（pull）一个镜像，Docker执行run方法得到一个容器，用户在容器里执行各种操作。Docker执行commit方法将一个容器转化为镜像。Docker利用login、push等命令将本地镜像推送（push）到仓库。其他机器或服务器上就可以使用该镜像去生成容器，进而运行相应的应用程序了。

## 6、利用Docker创建一个用于Flask开发的Python环境

```shell
$ docker pull centos
$ docker run -it centos:latest /bin/bash
## 此时进入容器，安装python3、git、flask及其依赖包,exit退出
$ docker commit -c "Flask" -a "gusuchen" container_id "gusuchen163/flask:v1"
$ docker push gusuchen163/flash:v1
```

## 7、Dockerfile 语法

#### 7-1. FROM

最佳实践：尽量使用官方的image作为 base image（为了安全）

```shell
FROM scratch       ## 制作 base image
FROM centos:latest ## 使用 base image
FROM ubuntu:14.04  ## 使用 base image
```

#### 7-2. LABEL

最佳实践：Metadata不可少

```shell
LABEL maintainer="gu_suchen@163.com"
LABEL version="1.0"
LABEL description="this is discription"
```

#### 7-3. RUN

最佳实践：

- 为了美观，复杂的RUN启用反斜线换行。
- 避免无用分层，合并多条命令成一行。

```shell
RUN yum update && yum install -y vim \
    python-dev  ## 反斜线换行
```

```shell
RUN apt-get update && apt-get install -y perl \
    pwgen --no-install-recommends && rm -rf \
    /var/lib/apt/list/*  ## 注意清理cache
```

```shell
RUN /bin/bash -c 'source $HOME/.bashrc;echo $HOME'
```

#### 7-4. WORKDIR

最佳实践：

- 用WORKDIR，不要用 RUN cd。
- 尽量使用绝对目录。

```shell
WORKDIR /root
```

```shell
WORKDIR /test  ## 如果没有自动创建目录test
WORKDIR demo
RUN pwd        ## 输出结果：/test/demo
```

#### 7-5. ADD and COPY

最佳实践：

- 大部分情况，COPY由于ADD。
- ADD除了COPY还有额外功能（解压）。
- 添加远程文件/目录，请使用curl或者wget。

ADD 不仅可以添加文件到指定目录，还可以解压缩

```
ADD hello /
```

```
ADD test.tar.gz /  ## 添加到根目录并解压
```

```
WORKDIR /root
ADD hello test/    ## /root/test/hello
```

```
WORKDIR /root
COPY hello test/
```

#### 7-6. ENV

最佳实践:尽量使用ENV增加可维护性

```shell
ENV MYSQL_VERSION 5.6  ## 设置常量
RUN apt-get install -y mysql-server="${MYSQL_VERSION}" \
    && rm -rf /var/lib/apt/lists/*  ## 引用常量
```

#### 7-7. VOLUME 和 EXPOSE

#### 7-8. CMD 和 ENTRYPOINT

- RUN CMD ENTRYPOINT 三者之间的区别

  - RUN: 执行命令并创建新的Image Layer
  - CMD: 设置容器启动后默认执行的命令和参数
  - ENTRYPOINT:设置容器启动后运行的命令

- Shell 和 Exec 两种格式

  - Shell 格式

    ```shell
    RUN centos
    ENV name Docker
    CMD echo "hello $name"
    ```

    ```shell
    RUN centos
    ENV name Docker
    ENTRYPOINT echo "hello $name"
    ```

  - Exec 格式

    ```shell
    RUN centos
    ENV name Docker
    CMD ["/bin/bash", "-c", "echo hello $name"]
    ```

    ```shell
    RUN centos
    ENV name Docker
    ENTRYPOINT ["/bin/bash", "-c", "echo hello $name"]
    ```

- CMD

  - 容器启动后默认执行的命令
  - 如果docker run 指定了其它命令，CMD命令会被忽略
  - 如果定义了多个CMD，只有最后一个会执行

  ```shell
  $ docker build -t gusuchen163/centos-cmd-shell .
  $ docker images
  $ docker run gusuchen163/centos-cmd-shell
  $ docker run gusuchen163/centos-cmd-shell /bin/bash
  ```

- ENTRYPOINT

  - 让容器易应用程序或者服务的形式运行

  - 不会被忽略，一定会执行

  - 最佳实践：写一个shell脚本作为entrypoint

    ```shell
    COPY docker-entrypoint.sh /usr/local/bin
    ENTRYPOINT ["docker-entrypoint.sh"]
    
    EXPOSE 27017
    CMD ["mongod"]
    ```

#### 7-9 Dockerfile 实践

- python程序

  ```shell
  from flask import Flask
  app = Flask(__name__)
  @app.route('/')
  def hello():
      return "hello docker"
  if __name__ == '__main__':
      app.run()
  ```

  

- Dockerfile

  ```shell
  FROM python:2.7
  LABEL maintainer="gusuchen<gu_suchen@163.com>"
  RUN pip install flask
  COPY app.py /app/
  WORKDIR /app
  EXPOSE 5000
  CMD ["python", "app.py"]
  ```

  

- build image 和 run container

  注意：构建镜像时，中间的image layer是可以运行docker run -it container_id进去的，方便定位问题

  ```shell
  $ docker build -t gusuchen163/flask-hello-world .  ## 构建image
  $ docker images
  $ docker run -d gusuchen163/flask-hello-world  ## -d 后台运行container
  $ docker ps -l
  $ docker stop gusuchen163/flask-hello-world    ## 停止容器
  $ docker start gusuchen163/flask-hello-world   ## 启动容器
  $ docker restart gusuchen163/flask-hello-world ## 重启容器
  $ docker attach gusuchen163/flask-hello-world  ## 进入运行中的容器
  ```

#### 7-10.Dockerfile使用

```bash
$ touch Dockerfile
$ vim Dockerfile
## 'FORM' 在某个基础镜像之上进行扩展<centos:latest；:latest是版本>
FORM centos
## 'MAINTAINER' 镜像创建者
MAINTAINER gusuchen gusuchen@outlook.com
## 'ADD' 添加 nginx-1.12.2.tar.gz 文件到 /usr/local/src
ADD nginx-1.12.2.tar.gz /usr/local/src
## 'EXPOSE' 镜像开放6379端口
EXPOSE 6379
##'ENTRYPOINT' 镜像启动时要执行的命令，必须是前台执行的方式，一般都是自己的应用系统启动命令
ENTRYPOINT java -version
## 'ENV' 添加环境变量
ENV JAVA_HOME /usr/lib/java-8

## 多条命令可使用 \ 换行，下一行使用 && 开头，整个Dockerfile最好只有一个RUN因为每个RUN都是一层镜像
RUN '创建镜像要执行的命令《比如安装软件》'
## 创建镜像，镜像完成后所在目录可使用 . 代表当前目录
docker build -t '镜像名称' '镜像完成后所在目录'
## 可存储密码，以及同时使用多个进程以及开启后台进程 <具体可 百度>
Supervisor docker
```

## 8、docker网络

#### 8-1 Linux网络命名空间（network namespace）

- 宿主机上新建两个docker容器，会自动的分配ip地址

  ```shell
  $ docker run -d --name test1 busybox /bin/sh -c 'whild true;do sleep 10; done'
  $ docker exec -it test1 /bin/sh
  / # ip link
  9: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
      link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
  / # ip a
  9: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
      link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
         valid_lft forever preferred_lft forever
  / # exit
  $ docker run -d --name test2 busybox /bin/sh -c 'while true;do sleep 10; done'
  $ docker exec -it test2 /bin/sh
  / # ip a
  13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
      link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.4/16 brd 172.17.255.255 scope global eth0
         valid_lft forever preferred_lft forever
  / # exit
  ```

  发现宿主机上的容器都链接上一个同一个网关：docker0，容易才可以相互连接

  ```shell
  $ ip a
  4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
      link/ether 02:42:ad:b5:5d:30 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
         valid_lft forever preferred_lft forever
      inet6 fe80::42:adff:feb5:5d30/64 scope link
         valid_lft forever preferred_lft forever
  10: veth69321b6@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
      link/ether 12:d7:1d:d5:14:d1 brd ff:ff:ff:ff:ff:ff link-netnsid 1
      inet6 fe80::10d7:1dff:fed5:14d1/64 scope link
         valid_lft forever preferred_lft forever
  14: vethe7b30da@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
      link/ether fe:0c:de:b2:1a:68 brd ff:ff:ff:ff:ff:ff link-netnsid 2
      inet6 fe80::fc0c:deff:feb2:1a68/64 scope link
         valid_lft forever preferred_lft forever
  ```

  

- 实践：模拟容易中网络的设置

  1. 宿主机上创建2个网络命名空间：test1、test2;
  2. 宿主机桑创建2个网络地址接口：veth-test1, veth-test2；
  3. 将veth-test1添加进test1，veth-test2添加进test2；
  4. veth-test1分配地址：192.168.1.1，veth-test2分配地址：192.168.1.2
  5. 分别启动网卡：veth-test1、veth-test2
  6. 相互进行ping，看是否连接

  ```shell
  $ sudo ip netns list   ## 查看网络命名空间
  $ sudo ip netns delete test1 ## 删除网络命名空间
  $ sudo ip netns ip link
  $ sudo ip netns ip a
  ## 1.宿主机上创建2个网络命名空间：test1、test2;
  $ sudo ip netns add test1  
  $ sudo ip netns add test2
  
  ## 2.宿主机上创建2个网络地址接口：veth-test1, veth-test2；
  $ sudo ip link add veth-test1 type veth peer name veth-test2
  
  ## 3.将veth-test1添加进test1，veth-test2添加进test2；
  $ sudo ip link set veth-test1 netns test1
  $ sudo ip link set veth-test2 netns test2
  
  ## 4.veth-test1分配地址：192.168.1.1，veth-test2分配地址：192.168.1.2
  $ sudo ip netns exec test1 ip addr add 192.168.1.1/24 dev veth-test1
  $ sudo ip netns exec test2 ip addr add 192.168.1.2/24 dev veth-test2
  
  ## 5.启动网卡：veth-test1、veth-test2
  $ sudo ip netns exec test1 ip link set dev veth-test1 up
  $ sudo ip netns exec test2 ip link set dev veth-test2 up
  
  ## 6.相互进行ping，看是否连接
  $ sudo ip netns exec test1 ping 192.168.1.2
  $ sudo ip netns exec test2 ping 192.168.1.1
  ```

#### 8-2 Docker Bridge0详解

- 俩个容器之间是如何通信的？

  发现 veth7ac3489@if30、vethbb07792@if36、veth17102c2@if48都链接到docker0

  ```shell
  $ ip link
  4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
      link/ether 02:42:ad:b5:5d:30 brd ff:ff:ff:ff:ff:ff
  31: veth7ac3489@if30: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
      link/ether da:ad:17:2b:93:4f brd ff:ff:ff:ff:ff:ff link-netnsid 0
  37: vethbb07792@if36: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
      link/ether c6:91:3d:2e:af:c9 brd ff:ff:ff:ff:ff:ff link-netnsid 1
  49: veth17102c2@if48: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
      link/ether 4a:bb:bc:1d:66:b6 brd ff:ff:ff:ff:ff:ff link-netnsid 6
  ```

  查看docker容器所有的网络？

  ```shell
  $ docker network ls
  NETWORK ID          NAME                DRIVER              SCOPE
  6bf283c4ca86        bridge              bridge              local
  8d6e47b7d1b0        host                host                local
  7a5b6480538f        my-net              bridge              local
  d191bc1ec726        none                null                local
  ```

  查看容器网关下面的容器网卡

  ```shell
  $ docker network inspect bridge
          "Containers": {
              "00ccb42d68c95ae1616eb1bb68dc3501e94efbad4a5d48ea00ab2dde91e11e6b": {
                  "Name": "friendly_mclaren",
                  "EndpointID": "3a862badbb0b3b2be0db86656909cfcc30dbe6a10333ae7e2e35d111d3f31110",
                  "MacAddress": "02:42:ac:11:00:03",
                  "IPv4Address": "172.17.0.3/16",
                  "IPv6Address": ""
              },
              "3ee7725a3f19dff3d9c518de0857186acc646555f0da88e298350b338a9d9bf7": {
                  "Name": "nexus3",
                  "EndpointID": "6f250bfcb6db94818000df4bc9be497b2a78571586dd5770a8aca1a1e3daac87",
                  "MacAddress": "02:42:ac:11:00:02",
                  "IPv4Address": "172.17.0.2/16",
                  "IPv6Address": ""
              },
              "59044b4be79b0bdfc80e963340d070dfaa270b7c39c2e5f4251c8b8760f008ce": {
                  "Name": "gracious_diffie",
                  "EndpointID": "e62862c6b5d3c7ad616c982239438c6464330bd906a080b63ab98d870fa2c7d9",
                  "MacAddress": "02:42:ac:11:00:04",
                  "IPv4Address": "172.17.0.4/16",
                  "IPv6Address": ""
              }
          },
  ```

  查看宿主机网关下面的容器网卡

  ```shell
  $ brctl show
  bridge name     bridge id               STP enabled     interfaces
  docker0         8000.0242adb55d30       no              veth17102c2
                                                          veth7ac3489
                                                          vethbb07792
  ```

- 容器如何与外网通信？

#### 8-3. 容器的端口映射

```shell
$ docker run --name web -d nginx
$ docker exec -it web /bin/bash
$ docker network ls
$ docker network inspect bridge
$ brctl show
$ telnet 172.17.0.3 80
$ curl 172.17.0.3
```

```shell
$ docker ps 
$ docker stop web
$ docker rm web
$ docker run --name web -d -p 80:80 nginx
$ docker ps -l
$ curl 127.0.0.2
$ curl 192.168.205.10
```

#### 8-4. 容器的网络：host/none

- **none** network：没有任何网络，不允许外部连接

  ```shell
  $ docker run -d --name busybox-none --network none busybox /bin/sh -c \ 
    > 'while true; do sleep 20 ; done;'
  $ docker network inspect none
  $ docker network ls
  $ docker exec -it busybox-none /bin/sh
  $ ip a
  ```

- **host** network：容器的网络和宿主机的网络一模一样，端口可能会有冲突

  ```shell
  $ docker run -d --name busybox-host --network host busybox /bin/sh -c \
    > 'while true; do sleep 20 ; done;'
  $ docker network ls
  $ docker network inspect host
  $ docker exec -it busybox-host /bin/sh
  $ ip a
  ```

#### 8-5. 多容器复杂应用的部署

#### 8-6. overlay和underlay的通俗解释

#### 8-7. dk overlay网络和etcd实现多机通信

## 9、Volume存储使用

```bash
## 将本机目录 /storage 挂载到 javad 容器的 /bin/bash目录 <注意：容器删除目录还在>
$ docker run --rm=true -it -v /storage /leader javad /bin/bash
## 和上面的命令一样只是加了 --privileged=true 可将多个镜像挂载到同一个目录，已达到文件共享
$ docker run --rm=true --privileged=true -it -v /storage /leader javad /bin/bash
## 和上面的命令一样只是加了 --volumes-from='容器ID'，将某个容器挂载到 javad 容器的 /bin/bash目录
$ docker run --rm=true --privileged=true --volumes-from='容器ID' -it javad /bin/bash
## 使用--link=127.0.0.1:myserver，将远程容器挂载到 javad 容器的 /bin/bash目录《myserver是远程容器在当前容器的一个别名，可使用 ping myserver 查看可ping通》
$ docker run --rm=true --link=127.0.0.1:myserver -it javad /bin/bash
## 容器共享同一个网络《mysqlserver=容器名称》，可用于多个服务使用同一个网络
$ docker run --rm=true --net=container:mysqlserver javad ip addr
## 查看文件所写的真实目录 《可直接在看到的目录下写数据》
$ docker inspect '容器ID'
```

## 10、Docker 路由机制打通网络《比较高效推荐使用》	

现在两个docker镜像128,130

```bash
修改128镜像：
    vi /usr/lib/systemd/system/docker.service
	ExecStart=/usr/bin/docker daemon --bip=172.18.42.1/16 -H fd:// -H=unix:///var/run/docker.sock
	systemctl daemon-reload
	重启128 docker 镜像
	
128镜像上执行 route add -net 172.17.0.0/16 gw 192.168.18.130
130镜像上执行 route add -net 172.18.0.0/16 gw 192.168.18.128

可以 ping 看看能不能通，如果不同可清理防火墙规则试试


注：有时间看看 docker + open vSwitch 打通网络
```

## 11、Docker-Compose<半个容器编排，还是用 k8s 吧> 可以在容器中直接使用 service 名称 代替 IP，相互访问容器里面的 service；使用如下

```bash
1，定义 docker-compose.yml 内容如下：
    version: '3'                                     -- docker compose 版本
	service:                                         -- service 定义
	  message-service:                               -- service 名称
	    image: message-service:latest                -- service 镜像名称
		ports: 
		- 8080:8080                                  -- service 对外提供的端口
      
      user-service:                            -- service 名称
        image: user-service:latest             -- service 镜像
        command                                -- 命令行参数
        - "--mysql.address=192.168.0.1"		    -- 参数 就是 user-service 容器在启动时所添加的命令行参数，在spring配置文件里面可使用${mysql.address}	取到
		
	  user-edge-service:                       -- service 名称             
	      image: user-edge-service: latest     -- 镜像名称
		  links:                                -- 依赖 service
		  - user-servive                        -- 依赖 service 名称
		  - message-service                     -- 依赖 service 名称
		  command:                              -- 命令行参数
		  - "--redis.address=127.0.0.1"         -- 参数 就是 user-service 容器在启动时所添加的命令行参数，在spring配置文件里面可使用${redis.address}	取到
		  
		  
2，docker-compose up -d                        -- 后台运行 docker-compose
```

## 12、附录

```bash
netstat -nlpt                                         --查看所有端口映射情况
netstat -nlpt |grep 3306                              --查看3306端口使用情况
service mysqld stop                                   --停止名叫 mysqld 的服务 
mysql -uroot -p                                       --centos7 使用mysql
env                                                   --centos7 查看环境变量
```

## 13、删除Docker

```bash
## 列出 docker 安装的软件包
$ yum list installed | grep docker
## 卸载 docker
$ yum -y remove '安装的软件包名'
## 删除 docker 镜像、容器，卷组和用户自配置文件。
$ rm -rf /var/lib/docker
```

## 14、说明

```bash
docker 默认支持互通，可通过 -icc=false 关闭互通。《/usr/bin/docker daemon --icc=false》
私有库搭建可使用：https://github.com/goharbor/harbor/releases	
```