# 一、预先准备环境

## 1. 准备服务器
这里准备了三台 centos 虚拟机，每台一核cpu和2G内存，配置好root账户，并安装好了docker，后续的所有操作都是使用root账户。虚拟机具体信息如下表：

| 系统类型 | IP地址 | 节点角色 | CPU | Memory | Hostname |
| :------: | :--------: | :-------: | :-----: | :---------: | :-----: |
| centos-64 | 192.168.1.101 | worker |   1    | 2G | server01 |
| centos-64 | 192.168.1.102 | master |   1    | 2G | server02 |
| centos-64 | 192.168.1.103 | worker |   1    | 2G | server03 |

## 2. 安装docker（所有节点）

#### 2.1 更新 yum 源

```bash
$ yum install -y wget
$ mkdir -p /etc/yum.repos.d/bak
$ mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak/
```

```bash
## 更新国内yum源
$ wget -O /etc/yum.repos.d/Centos-Base.repo \
http://mirrors.cloud.tencent.com/repo/centos7_base.repo

## 更新国内yum源
$ wget -O /etc/yum.repos.d/epel.repo http://mirrors.cloud.tencent.com/repo/epel-7.repo
```
```bash
$ yum clean all && yum makecache
$ yum install -y git vim gcc glibc-static telnet bridge-utils net-tools
```

#### 2.2 添加 docker.repo

```bash
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo \
-O /etc/yum.repos.d/docker-ce.repo
```

#### 2.3 添加 kubernetes.repo

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
```

```bash
$ yum clean all && yum makecache
```

#### 2.4 安装 docker

[docker安装]: https://docs.docker.com/install/linux/docker-ce/centos/

```bash
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
$ sudo yum install docker-ce docker-ce-cli containerd.io -y
$ systemctl enable docker && systemctl start docker
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
配置阿里云docker加速器

```bash
$ sudo mkdir -p /etc/docker
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://741gqwq9.mirror.aliyuncs.com"]
}
EOF
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```
#### 2.5 配置docker

- 配置所有ip的数据包转发
```bash
$ vim /lib/systemd/system/docker.service
#找到ExecStart=xxx，在这行下面加入一行，内容如下：(k8s的网络需要)
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
```
- 启动服务
```bash
## 设置 docker 开机服务启动
$ systemctl enable docker.service 
## 立即启动 docker 服务
$ systemctl daemon-reload && systemctl restart docker
```


遇到问题可以参考：https://docs.docker.com/install/linux/docker-ce/centos/

## 3. 系统设置（所有节点）
#### 3.1 关闭、禁用防火墙(让所有机器之间都可以通过任意端口建立连接)
```bash
## 查看防火墙状态
$ firewall-cmd --state
## 关闭防火墙
$ systemctl stop firewalld.service
## 禁用防火墙<开机不会自动启动>
$ systemctl disable firewalld.service
```
#### 3.2 设置系统参数 - 允许路由转发，不对bridge的数据进行处理
```bash
## 写入配置文件
$ tee /etc/sysctl.d/k8s.conf <<-'EOF'
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
 
## 生效配置文件
$ sysctl -p /etc/sysctl.d/k8s.conf
```

#### 3.3 配置host文件
```bash
## 配置host，使每个Node都可以通过名字解析到ip地址
$ vim /etc/hosts
## 加入如下片段(ip地址和servername替换成自己的) 本机不需要配置 hosts
192.168.1.101 server01
192.168.1.102 server02
192.168.1.103 server03
192.168.1.104 server04
192.168.1.105 server05
```

#### 3.4 安装jdk和maven

```bash
$ tar -zxvf /home/vagrant/labs/jdk-8u144-linux-x64.tar.gz -C /usr/local/
$ tar -zxvf /home/vagrant/labs/apache-maven-3.5.4-bin.tar.gz -C /usr/local/
$ vim /etc/profile
export JAVA_HOME=/usr/local/jdk1.8.0_144
export MAVEN_HOME=/usr/local/apache-maven-3.5.4
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin

$ java -version
$ mvn -version
```



## 4. 准备二进制文件（所有节点）

####[下载地址（kubernetes 1.9.0版本）][2] 或者 自己上 github 下载最新版本

```bash
#下载后找个目录解压
tar -zxvf /home/vagrant/labs/kubernetes-bins.tar.gz -C /home/kubernetes/
#将解压后的文件名改为bin
mv /home/kubernetes/kubernetes-bins/ /home/kubernetes/bin
#配置环境变量
vim ~/.bashrc 或者 vim /etc/profile
#添加如下内容
export KUBERNETES_HOME=/home/kubernetes
export PATH=${KUBERNETES_HOME}/bin:$PATH
#确定修改
source ~/.bashrc
#测试 kubernetes环境变量是否配置成功
kubectl -h
```

## 5. 准备配置文件（所有节点）
上一步我们下载了kubernetes各个组件的二进制文件，这些可执行文件的运行也是需要添加很多参数的，包括有的还会依赖一些配置文件。现在我们就把运行它们需要的参数和配置文件都准备好。
#### 5.1 下载配置文件
```bash
#到home目录下载项目(git最好只装一台)
$ yum -y install git
$ cd
$ git clone https://github.com/chiangfire/kubernetes-starter.git
#看看git内容
$ cd ~/kubernetes-starter && ls
```

#### 5.2 文件说明

- **gen-config.sh**
> shell脚本，用来根据每个同学自己的集群环境(ip，hostname等)，根据下面的模板，生成适合大家各自环境的配置文件。生成的文件会放到target文件夹下。

- **kubernetes-simple**
> 简易版kubernetes配置模板（剥离了认证授权）。
> 适合刚接触kubernetes的同学，首先会让大家在和kubernetes初次见面不会印象太差（太复杂啦~~），再有就是让大家更容易抓住kubernetes的核心部分，把注意力集中到核心组件及组件的联系，从整体上把握kubernetes的运行机制。

- **kubernetes-with-ca**
> 在simple基础上增加认证授权部分。大家可以自行对比生成的配置文件，看看跟simple版的差异，更容易理解认证授权的（认证授权也是kubernetes学习曲线较高的重要原因）

- **service-config**
>这个先不用关注，它是我们曾经开发的那些微服务配置。
> 等我们熟悉了kubernetes后，实践用的，通过这些配置，把我们的微服务都运行到kubernetes集群中。

#### 5.3 生成配置
这里会根据大家各自的环境生成kubernetes部署过程需要的配置文件。
在每个节点上都生成一遍，把所有配置都生成好，后面会根据节点类型去使用相关的配置。
```bash
#cd到之前下载的git代码目录
$ cd ~/kubernetes-starter
#编辑属性配置（根据文件注释中的说明填写好每个key-value）
$ vi config.properties
#生成配置文件，确保执行过程没有异常信息
$ ./gen-config.sh simple
#查看生成的配置文件，确保脚本执行成功
$ find target/ -type f
target/all-node/kube-calico.service
target/master-node/kube-controller-manager.service
target/master-node/kube-apiserver.service
target/master-node/etcd.service
target/master-node/kube-scheduler.service
target/worker-node/kube-proxy.kubeconfig
target/worker-node/kubelet.service
target/worker-node/10-calico.conf
target/worker-node/kubelet.kubeconfig
target/worker-node/kube-proxy.service
target/services/kube-dns.yaml
```
> **执行gen-config.sh常见问题：**
> 1. gen-config.sh: 3: gen-config.sh: Syntax error: "(" unexpected
> - bash版本过低，运行：bash -version查看版本，如果小于4需要升级
> - 不要使用 sh gen-config.sh的方式运行（sh和bash可能不一样哦）
> 2. config.properties文件填写错误，需要重新生成
> 再执行一次./gen-config.sh simple即可，不需要手动删除target

[1]: https://docs.docker.com/install/linux/docker-ce/centos/
[2]: https://pan.baidu.com/s/1bMnqWY	"kubernetes-1.9.0版本"
