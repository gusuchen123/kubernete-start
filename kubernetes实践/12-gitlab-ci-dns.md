## 一、GitLab-CI DNS Server（Docker版）

## 二、

## 一、下载镜像

官方版本是：gitlab/gitlab-runner:latest

```bash
## 官方镜像，从hub.docker.com拉取
$ docker pull gitlab/gitlab-runner
Using default tag: latest
latest: Pulling from gitlab/gitlab-runner
35b42117c431: Already exists
ad9c569a8d98: Already exists
293b44f45162: Already exists
0c175077525d: Already exists
18d6937c67f1: Pull complete
310b87723509: Pull complete
1c8f4344f06d: Pull complete
35515902a4bc: Pull complete
f02161bfe466: Pull complete
Digest: sha256:20218167297797ac38aaa7b7bcae78997981fecc685ba0208813cec1e0cc9047
Status: Downloaded newer image for gitlab/gitlab-runner:latest
$ docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
gitlab/gitlab-runner               latest              4142c6fc05d4        5 days ago          410MB
gitlab/gitlab-ce                   latest              15563c211d40        10 days ago         1.8GB
```

## 二、运行DNS-Server容器

#### 1.关闭selinux

```bash
[root@server05 gitlab-runner]# getenforce
Enforcing
[root@server05 gitlab-runner]# setenforce 0
[root@server05 gitlab-runner]# getenforce
Permissive
[root@server05 gitlab-runner]# vim /etc/sysconfig/selinux
SELINUX=disabled
```



## 三、DNS-Server使用