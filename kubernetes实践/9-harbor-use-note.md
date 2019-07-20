## 一、公有仓库 `DockerHub`

### 1. 在`DockerHub`注册账户

```bash
https://hub.docker.com/
```

### 2.使用`DockerHub`

- 登录

  ```bash
  $ docker login
  WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
  Configure a credential helper to remove this warning. See
  https://docs.docker.com/engine/reference/commandline/login/#credentials-store
  
  Login Succeeded
  ```

- 上传镜像

  ```bash
  $ docker tag zookeeper:3.5 gusuchen163/zookeeper:3.5
  $ docker push gusuchen163/zookeeper:3.5
  The push refers to repository [docker.io/gusuchen163/zookeeper]
  12bfba8df297: Mounted from library/zookeeper
  5324becb6741: Mounted from library/zookeeper
  7902a20cfb21: Mounted from library/zookeeper
  93a28b6faa6a: Mounted from library/zookeeper
  99bd4bc6a009: Mounted from library/zookeeper
  2769275ecc2d: Mounted from library/zookeeper
  9ecd51c60835: Mounted from library/zookeeper
  cf5b3c6798f7: Mounted from library/zookeeper
  3.5: digest: sha256:c94f1a49a33df8b7c3bbe9793ca1e0980dc599f22c10b6eda425365efc943fa3 size: 1998
  ```

- 下载镜像

  ```bash
  $ docker pull gusuchen163/zookeeper:3.5
  3.5: Pulling from gusuchen163/zookeeper
  Digest: sha256:c94f1a49a33df8b7c3bbe9793ca1e0980dc599f22c10b6eda425365efc943fa3
  Status: Image is up to date for gusuchen163/zookeeper:3.5
  ```

## 二·、私有镜像仓库`registry`和`harbor`

### 1.使用`registry`

- 从`DockerHub`拉取镜像`resgitry`，编写`start.sh`

  ```bash
  $ tee start.sh <<-'EOF'
  #!/bin/bash
  CUR_DIR=`pwd`
  docker pull registry:2
  docker stop registry
  docker rm registry
  docker run -d --name registry --restart always \
         -p 5000:5000 \
         -v ${CUR_DIR}/registry:/var/lib/registry \
         registry:2
  EOF
  ```

  ```bash
  $ sh start.sh
  2: Pulling from library/registry
  Digest: sha256:8004747f1e8cd820a148fb7499d71a76d45ff66bac6a29129bfdbfdc0154d146
  Status: Downloaded newer image for registry:2
  registry
  registry
  cf06bfecd3940316b97d7042455af6dc9f165557a156a3da9589dbd57b6ecff3
  $ docker ps
  CONTAINER ID        IMAGE          COMMAND        CREATED          STATUS         PORTS    NAMES
  cf06bfecd394        registry:2                      "/entrypoint.sh /etc…"   About a minute ago   Up About a minute               0.0.0.0:5000->5000/tcp                                 registry
  $ docker logs -f registry
  time="2019-07-13T12:52:02.698730416Z" level=warning msg="No HTTP secret provided - generated random secret. This may cause problems with uploads if multiple registries are behind a load-balancer. To provide a shared secret, fill in http.secret in the configuration file or set the REGISTRY_HTTP_SECRET environment variable." go.version=go1.11.2 instance.id=08c62d82-6dc2-49b5-a2f5-04817d6f6d92 service=registry version=v2.7.1
  time="2019-07-13T12:52:02.698831964Z" level=info msg="redis not configured" go.version=go1.11.2 instance.id=08c62d82-6dc2-49b5-a2f5-04817d6f6d92 service=registry version=v2.7.1
  $ echo "192.168.1.104 registry.gusuchen.com" >> /etc/hosts
  $ cat /etc/hosts
  127.0.0.1       server04        server04
  127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
  ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
  
  192.168.1.101 server01
  192.168.1.102 server02
  192.168.1.103 server03
  192.168.1.104 server04
  192.168.1.105 server05
  
  192.168.1.104 hub.gusuchen.com
  192.168.1.104 registry.gusuchen.com
  ```

- 上传镜像和下载镜像到`registry`

  ```bash
  $ docker pull ubuntu
  Using default tag: latest
  latest: Pulling from library/ubuntu
  5b7339215d1d: Pull complete
  14ca88e9f672: Pull complete
  a31c3b1caad4: Pull complete
  b054a26005b7: Pull complete
  Digest: sha256:9b1702dcfe32c873a770a32cfd306dd7fc1c4fd134adfb783db68defc8894b3c
  Status: Downloaded newer image for ubuntu:latest
  $ docker images
  REPOSITORY              TAG                 IMAGE ID            CREATED         SIZE
  gusuchen163/redis       5.0.5               4540404d671d        3 hours ago     95.1MB
  redis                   5.0.5               bb0ab8a99fe6        4 days ago      95MB
  ubuntu                  latest              4c108a37151f        2 weeks ago     64.2MB
  gusuchen163/zookeeper   3.5                 215d317d188b        3 weeks ago     211MB
  zookeeper               3.5                 215d317d188b        3 weeks ago     211MB
  mysql                   5.7.26              a1aa4f76fab9        3 weeks ago     373MB
  registry                2.7                 f32a97de94e1        4 months ago    25.8MB
  $ docker tag ubuntu localhost:5000/ubuntu:latest
  $ docker push localhost:5000/ubuntu:latest
  The push refers to repository [localhost:5000/ubuntu]
  75e70aa52609: Pushed
  dda151859818: Pushed
  fbd2732ad777: Pushed
  ba9de9d8475e: Pushed
  latest: digest: sha256:eb70667a801686f914408558660da753cde27192cd036148e58258819b927395 size: 1152
  $ docker pull localhost:5000/ubuntu:latest
  latest: Pulling from ubuntu
  Digest: sha256:eb70667a801686f914408558660da753cde27192cd036148e58258819b927395
  Status: Image is up to date for localhost:5000/ubuntu:latest
  $ docker tag zookeeper:3.5 registry.gusuchen.com:5000/zookeeper:3.5
  $ docker push registry.gusuchen.com:5000/zookeeper:3.5
  ```

- 缺点：

  - 单点问题
  - 没有友好的操作界面

### 2.使用`harbor`

参考博客：https://www.jianshu.com/p/9ba8807d33cc

- 前提条件
  - docker 服务是否已安装，并且处于 Running 状态；
  - docker-compose是否已经安装；

- 下载并安装docker-compose(官网推荐)

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

- 下载并修改配置文件 `habor.yml`

  ```bash
  ## 下载安装包
  $ wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.1.tgz
  ## 解压 harbor-offline-installer-v1.8.1.tgz
  $ tar -zxvf harbor-offline-installer-v1.8.1.tgz
  $ ls
  harbor.v1.8.1.tar.gz  harbor.yml  install.sh  LICENSE  prepare
  ## 修改配置文件：hostname和port
  $ vim harbor.yml
  hostname: hub.gusuchen.com
  # http related config
  http:
    # port for http, default is 80. If https enabled, this port will redirect to https port
    port: 80
  ```

- 修改文件：`/usr/lib/systemd/system/docker.service`

  `vim  /usr/lib/systemd/system/docker.service`

  ```bash
  ## 追加参数 --insecure-registry=hub.gusuchen.com
  ExecStart=/usr/bin/dockerd \
  -H fd:// --containerd=/run/containerd/containerd.sock \
  --insecure-registry=hub.gusuchen.com
  ```

  重新启动docker，是配置文件生效

  ```bash
  ## 重新加载配置文件，使生效
  $ systemctl daemon-reload && systemctl restart docker
  ```

- 执行安装脚本：`sh install.sh`

- 用户的初始化用户名和密码请查看`harbor.yml`

  ```
  admin / Harbor12345
  ```

- 使用浏览器访问：http://hub.gusuchen.com/，新建项目：test-harbor

- 镜像推送

  修改映射文件：`/etc/hosts`

  ```bash
  127.0.0.1       server04        server04
  127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
  ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
  
  192.168.1.101 server01
  192.168.1.102 server02
  192.168.1.103 server03
  192.168.1.104 server04
  192.168.1.105 server05
  
  192.168.1.104 hub.gusuchen.com
  ```

  ```bash
  ## 先随机拉取一个镜像，例如 nginx镜像
  $ docker pull docker.io/nginx:latest
  latest: Pulling from library/nginx
  fc7181108d40: Already exists
  d2e987ca2267: Pull complete
  0b760b431b11: Pull complete
  Digest: sha256:96fb261b66270b900ea5a2c17a26abbfabe95506e73c3a3c65869a6dbe83223a
  Status: Downloaded newer image for nginx:latest
  ## 将 nginx 镜像打上新的 tag，
  ## 格式是：${harbor 服务器 IP}：${harbor 监听端口}/${项目}/${镜像名称}：${镜像 tag}
  ## 确保项目(test-harbor)要在 harbor 上提前建好。
  $ docker tag docker.io/nginx hub.gusuchen.com/test-harbor/nginx:latest
  ## 登录harbor
  $ docker login hub.gusuchen.com
  Username: admin
  Password:
  WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
  Configure a credential helper to remove this warning. See
  https://docs.docker.com/engine/reference/commandline/login/#credentials-store
  
  Login Succeeded
  ## 推送镜像
  $ docker push hub.gusuchen.com/test-harbor/nginx:latest
  The push refers to repository [hub.gusuchen.com/test-harbor/nginx]
  d2f0b6dea592: Pushed
  197c666de9dd: Pushed
  cf5b3c6798f7: Pushed
  latest: digest: sha256:00be67d6ba53d5318cd91c57771530f5251cfbe028b7be2c4b70526f988cfc9f size: 948
  ```

  使用浏览器访问：http://hub.gusuchen.com/，查看上传的镜像

  注意：修改本地的hosts文件

  ```bash
  192.168.1.104  hub.gusuchen.com
  ```

- 启动|停止|重启|查看

  ```bash
  ## 后台启动
  $ docker-compose up -d
  ## 停止
  $ docker-compose stop
  ## 重启
  $ docker-compose restart
  ## 查看服务容器
  $ docker-compose ps
  ```

  

- 问题1

  ```bash
  问题描述：
  $ docker login hub.gusuchen.com
  Error response from daemon: Get https://hub.gusuchen.com/v2/: dial tcp 192.168.1.104:443: connect: connection refused
  ```

  ```bash
  ## 追加参数：--insecure-registry=hub.gusuchen.com
  $ vim /usr/lib/systemd/system/docker.service
  ExecStart=/usr/bin/dockerd \
  -H fd:// --containerd=/run/containerd/containerd.sock \
  --insecure-registry=hub.gusuchen.com
  
  ## 重新加载配置文件，使生效
  $ systemctl daemon-reload
  $ systemctl restart docker
  $ docker info | grep Insecure
  Insecure Registries:
  ## 启动Harbor
  $ cd harbor/
  ## 停止Harbor
  $ docker-compose down -v
  ## 后台启动Harbor
  $ docker-compose up -d
  ## 登录
  $ docker login hub.gusuchen.com
  ```

  

