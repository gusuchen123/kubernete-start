# 二、基础集群部署 - kubernetes-simple
## 1. 部署ETCD（主节点）
#### 1.1 简介
&emsp;&emsp;kubernetes需要存储很多东西，像它本身的节点信息，组件信息，还有通过kubernetes运行的pod，deployment，service等等。都需要持久化。etcd就是它的数据中心。生产环境中为了保证数据中心的高可用和数据的一致性，一般会部署最少三个节点。我们这里以学习为主就只在主节点部署一个实例。
> 如果你的环境已经有了etcd服务(不管是单点还是集群)，可以忽略这一步。前提是你在生成配置的时候填写了自己的etcd endpoint哦~

#### 1.2 部署
**etcd的二进制文件和服务的配置我们都已经准备好，现在的目的就是把它做成系统服务并启动。**

```bash
#把服务配置文件copy到系统服务目录
[root@server02 kubernetes-starter]# cp ./target/master-node/etcd.service \
> /lib/systemd/system/
#enable服务
[root@server02 kubernetes-starter]# systemctl enable etcd.service
#创建工作目录(保存数据的地方)
[root@server02 kubernetes-starter]# mkdir -p /var/lib/etcd
# 启动服务
[root@server02 kubernetes-starter]# service etcd start
# 查看服务日志，看是否有错误信息，确保服务正常
[root@server02 kubernetes-starter]# journalctl -f -u etcd.service
-- Logs begin at Sat 2019-07-06 02:08:49 UTC. --
Jul 06 05:10:44 server02 etcd[29761]: published {Name:192.168.1.102 ClientURLs:[http://192.168.1.102:2379]} to cluster cdf818194e3a8c32
Jul 06 05:10:44 server02 etcd[29761]: ready to serve client requests
Jul 06 05:10:44 server02 etcd[29761]: dialing to target with scheme: ""
Jul 06 05:10:44 server02 etcd[29761]: could not get resolver for scheme: ""
Jul 06 05:10:44 server02 etcd[29761]: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
Jul 06 05:10:44 server02 etcd[29761]: ready to serve client requests
Jul 06 05:10:44 server02 etcd[29761]: dialing to target with scheme: ""
Jul 06 05:10:44 server02 etcd[29761]: could not get resolver for scheme: ""
Jul 06 05:10:44 server02 etcd[29761]: serving insecure client requests on 192.168.1.102:2379, this is strongly discouraged!
Jul 06 05:10:44 server02 systemd[1]: Started Etcd Server.

[root@server02 kubernetes-starter]# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 192.168.1.102:2379      0.0.0.0:*              LISTEN      29761/etcd
tcp        0      0 127.0.0.1:2379          0.0.0.0:*              LISTEN      29761/etcd
tcp        0      0 127.0.0.1:2380          0.0.0.0:*              LISTEN      29761/etcd
```

## 2. 部署APIServer（主节点）
#### 2.1 简介
kube-apiserver是Kubernetes最重要的核心组件之一，主要提供以下的功能
- 提供集群管理的REST API接口，包括认证授权（我们现在没有用到）数据校验以及集群状态变更等
- 提供其他模块之间的数据交互和通信的枢纽（其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd）

> 生产环境为了保证apiserver的高可用一般会部署2+个节点，在上层做一个lb做负载均衡，比如haproxy。由于单节点和多节点在apiserver这一层说来没什么区别，所以我们学习部署一个节点就足够啦

#### 2.2 部署
APIServer的部署方式也是通过系统服务。部署流程跟etcd完全一样，不再注释
```bash
[root@server02 kubernetes-starter]# cp ./target/master-node/kube-apiserver.service \
> /lib/systemd/system/
[root@server02 kubernetes-starter]# systemctl enable kube-apiserver.service
[root@server02 kubernetes-starter]# service kube-apiserver start
[root@server02 kubernetes-starter]# journalctl -f -u kube-apiserver
[root@server02 kubernetes-starter]# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 192.168.1.102:2379      0.0.0.0:*              LISTEN      29761/etcd
tcp        0      0 127.0.0.1:2379          0.0.0.0:*              LISTEN      29761/etcd
tcp        0      0 127.0.0.1:2380          0.0.0.0:*              LISTEN      29761/etcd
tcp6       0      0 :::6443                 :::*                   LISTEN      1216/kube-apiserver
tcp6       0      0 :::8080                 :::*                    LISTEN      1216/kube-apiserver
```

#### 2.3 重点配置说明
> [Unit]  
> Description=Kubernetes API Server  
> ...  
> [Service]  
> \#可执行文件的位置  
> ExecStart=/home/michael/bin/kube-apiserver \\  
> \#非安全端口(8080)绑定的监听地址 这里表示监听所有地址  
> --insecure-bind-address=0.0.0.0 \\  
> \#不使用https  
> --kubelet-https=false \\  
> \#kubernetes集群的虚拟ip的地址范围  
> --service-cluster-ip-range=10.68.0.0/16 \\  
> \#service的nodeport的端口范围限制  
>   --service-node-port-range=20000-40000 \\  
> \#很多地方都需要和etcd打交道，也是唯一可以直接操作etcd的模块  
>   --etcd-servers=http://192.168.1.102:2379 \\  
> ...  

```
问题描述：连不上etcd
Jul 06 05:31:12 server02 kube-apiserver[30473]: error waiting for etcd connection: unable to parse etcd url: parse 192.168.1.102:2379: first path segment in URL cannot contain colon
Jul 06 05:31:12 server02 systemd[1]: kube-apiserver.service: main process exited, code=exited, status=1/FAILURE
Jul 06 05:31:12 server02 systemd[1]: Failed to start Kubernetes API Server.
Jul 06 05:31:12 server02 systemd[1]: Unit kube-apiserver.service entered failed state.
Jul 06 05:31:12 server02 systemd[1]: kube-apiserver.service failed.
```

```
解决办法：修改配置文件--/lib/systemd/system/kube-apiserver.service
--etcd-servers=http://192.168.1.102:2379 \\  
```

## 3. 部署ControllerManager（主节点）

#### 3.1 简介
Controller Manager由kube-controller-manager和cloud-controller-manager组成，是Kubernetes的大脑，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态。
kube-controller-manager由一系列的控制器组成，像Replication Controller控制副本，Node Controller节点控制，Deployment Controller管理deployment等等
cloud-controller-manager在Kubernetes启用Cloud Provider的时候才需要，用来配合云服务提供商的控制
> controller-manager、scheduler和apiserver 三者的功能紧密相关，一般运行在同一个机器上，我们可以把它们当做一个整体来看，所以保证了apiserver的高可用即是保证了三个模块的高可用。也可以同时启动多个controller-manager进程，但只有一个会被选举为leader提供服务。

#### 3.2 部署
**通过系统服务方式部署**
```bash
$ cp ./target/master-node/kube-controller-manager.service /lib/systemd/system/
$ systemctl enable kube-controller-manager.service
$ service kube-controller-manager start
$ journalctl -f -u kube-controller-manager
```
#### 3.3 重点配置说明
> [Unit]  
> Description=Kubernetes Controller Manager  
> ...  
> [Service]  
> ExecStart=/home/michael/bin/kube-controller-manager \\  
> \#对外服务的监听地址，这里表示只有本机的程序可以访问它  
>   --address=127.0.0.1 \\  
>   \#apiserver的url  
>   --master=http://127.0.0.1:8080 \\  
>   \#服务虚拟ip范围，同apiserver的配置  
>  --service-cluster-ip-range=10.68.0.0/16 \\  
>  \#pod的ip地址范围  
>  --cluster-cidr=172.20.0.0/16 \\  
>  \#下面两个表示不使用证书，用空值覆盖默认值  
>  --cluster-signing-cert-file= \\  
>  --cluster-signing-key-file= \\  
> ...  

## 4. 部署Scheduler（主节点）
#### 4.1 简介
kube-scheduler负责分配调度Pod到集群内的节点上，它监听kube-apiserver，查询还未分配Node的Pod，然后根据调度策略为这些Pod分配节点。我们前面讲到的kubernetes的各种调度策略就是它实现的。

#### 4.2 部署
**通过系统服务方式部署**
```bash
$ cp ./target/master-node/kube-scheduler.service /lib/systemd/system/
$ systemctl enable kube-scheduler.service
$ service kube-scheduler start
$ journalctl -f -u kube-scheduler
```

#### 4.3 重点配置说明
> [Unit]  
> Description=Kubernetes Scheduler  
> ...  
> [Service]  
> ExecStart=/home/michael/bin/kube-scheduler \\  
>  \#对外服务的监听地址，这里表示只有本机的程序可以访问它  
>   --address=127.0.0.1 \\  
>   \#apiserver的url  
>   --master=http://127.0.0.1:8080 \\  
> ...  

## 5. 部署CalicoNode（所有节点）
#### 5.1 简介
Calico实现了CNI接口，是kubernetes网络方案的一种选择，它一个纯三层的数据中心网络方案（不需要Overlay），并且与OpenStack、Kubernetes、AWS、GCE等IaaS和容器平台都有良好的集成。
Calico在每一个计算节点利用Linux Kernel实现了一个高效的vRouter来负责数据转发，而每个vRouter通过BGP协议负责把自己上运行的workload的路由信息像整个Calico网络内传播——小规模部署可以直接互联，大规模下可通过指定的BGP route reflector来完成。 这样保证最终所有的workload之间的数据流量都是通过IP路由的方式完成互联的。

#### 5.2 部署
**calico是通过系统服务+docker方式完成的**
```bash
$ cp ./target/all-node/kube-calico.service /lib/systemd/system/
$ systemctl enable kube-calico.service
$ service kube-calico start
$ journalctl -f -u kube-calico
```
#### 5.3 calico可用性验证
**查看容器运行情况（可能没有那么快）**

```bash
$ docker ps
CONTAINER ID   IMAGE                COMMAND        CREATED ...
4d371b58928b   calico/node:v2.6.2   "start_runit"  3 hours ago...
```
**安装 calicoctl(可以不安装)**
```bash
wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calicoctl/releases/download/v3.3.0/calicoctl   -- 下载安装包
chmod +x /usr/local/bin/calicoctl                                                                                -- 安装
```
**查看节点运行情况**
```bash
$ calicoctl node status
Calico process is running.
IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.1.103 | node-to-node mesh | up    | 13:13:13 | Established |
+---------------+-------------------+-------+----------+-------------+
IPv6 BGP status
No IPv6 peers found.
```
**查看端口BGP 协议是通过TCP 连接来建立邻居的，因此可以用netstat 命令验证 BGP Peer**

```bash
$ netstat -natp|grep ESTABLISHED|grep 179
tcp        0      0 192.168.1.102:60959     192.168.1.103:179       ESTABLISHED 29680/bird
```
**查看集群ippool情况  （因为我们使用k8s去控制所以可以不配置 ipPool 网络资源，如果要配置 calicoctl "ipPool" 网络资源再查查）**

```bash
$ calicoctl get ipPool -o yaml
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 172.20.0.0/16
  spec:
    nat-outgoing: true
```
#### 5.4 重点配置说明
> [Unit]  
> Description=calico node  
> ...  
> [Service]  
> \#以docker方式运行  
> ExecStart=/usr/bin/docker run --net=host --privileged --name=calico-node \\  
> \#指定etcd endpoints（这里主要负责网络元数据一致性，确保Calico网络状态的准确性）  
>   -e ETCD_ENDPOINTS=http://192.168.1.102:2379 \\  
> \#网络地址范围（同上面ControllerManager）  
>   -e CALICO_IPV4POOL_CIDR=172.20.0.0/16 \\  
> \#镜像名，为了加快大家的下载速度，镜像都放到了阿里云上  
>   registry.cn-hangzhou.aliyuncs.com/imooc/calico-node:v2.6.2  

## 6. 配置kubectl命令（任意节点）
#### 6.1 简介
kubectl是Kubernetes的命令行工具，是Kubernetes用户和管理员必备的管理工具。
kubectl提供了大量的子命令，方便管理Kubernetes集群中的各种功能。
#### 6.2 初始化
使用kubectl的第一步是配置Kubernetes集群以及认证方式，包括：
- cluster信息：api-server地址
- 用户信息：用户名、密码或密钥
- Context：cluster、用户信息以及Namespace的组合

我们这没有安全相关的东西，只需要设置好api-server和上下文就好啦：
```bash
#指定apiserver地址（ip替换为你自己的api-server地址）
$ kubectl config set-cluster kubernetes  --server=http://192.168.1.102:8080
#指定设置上下文，指定cluster
$ kubectl config set-context kubernetes --cluster=kubernetes
#选择默认的上下文
$ kubectl config use-context kubernetes
#测试kubectl，获取所有的pod（还没有开始使用应该是没有pod的）
apiVersion: v1
clusters:
- cluster:
    server: http://192.168.1.102:8080
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: ""
  name: kubernetes
current-context: kubernetes
kind: Config
preferences: {}
users: []

$ kubectl get pods
```
> 通过上面的设置最终目的是生成了一个配置文件：~/.kube/config，当然你也可以手写或复制一个文件放在那，就不需要上面的命令了。

## 7. 配置kubelet（工作节点）
#### 7.1 简介
每个工作节点上都运行一个kubelet服务进程，默认监听10250端口，接收并执行master发来的指令，管理Pod及Pod中的容器。每个kubelet进程会在API Server上注册节点自身信息，定期向master节点汇报节点的资源使用情况，并通过cAdvisor监控节点和容器的资源。
#### 7.2 部署
**通过系统服务方式部署，但步骤会多一些，具体如下：**
```bash
#确保相关目录存在
$ mkdir -p /var/lib/kubelet
$ mkdir -p /etc/kubernetes
$ mkdir -p /etc/cni/net.d

#复制kubelet服务配置文件
$ cp target/worker-node/kubelet.service /lib/systemd/system/
#复制kubelet依赖的配置文件
$ cp target/worker-node/kubelet.kubeconfig /etc/kubernetes/
#复制kubelet用到的cni插件配置文件
$ cp target/worker-node/10-calico.conf /etc/cni/net.d/

$ systemctl enable kubelet.service
$ service kubelet start
$ journalctl -f -u kubelet
```
#### 7.3 重点配置说明
**kubelet.service**
> [Unit]  
Description=Kubernetes Kubelet  
[Service]  
\#kubelet工作目录，存储当前节点容器，pod等信息  
WorkingDirectory=/var/lib/kubelet  
ExecStart=/home/michael/bin/kubelet \\  
  \#对外服务的监听地址  
  --address=192.168.1.103 \\  
  \#指定基础容器的镜像，负责创建Pod 内部共享的网络、文件系统等，这个基础容器非常重要：K8S每一个运行的 POD里面必然包含这个基础容器，如果它没有运行起来那么你的POD 肯定创建不了  
  --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/imooc/pause-amd64:3.0 \\  
  \#访问集群方式的配置，如api-server地址等  
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\  
  \#声明cni网络插件  
  --network-plugin=cni \\  
  \#cni网络配置目录，kubelet会读取该目录下得网络配置  
  --cni-conf-dir=/etc/cni/net.d \\  
  \#指定 kubedns 的 Service IP(可以先分配，后续创建 kubedns 服务时指定该 IP)，--cluster-domain 指定域名后缀，这两个参数同时指定后才会生效  
 --cluster-dns=10.68.0.2 \\  
  ...  

**kubelet.kubeconfig**  
kubelet依赖的一个配置，格式看也是我们后面经常遇到的yaml格式，描述了kubelet访问apiserver的方式
> apiVersion: v1  
> clusters:  
> \- cluster:  
> \#跳过tls，即是kubernetes的认证  
>     insecure-skip-tls-verify: true  
>   \#api-server地址  
>     server: http://192.168.1.102:8080  
> ...  

**10-calico.conf**  
calico作为kubernets的CNI插件的配置
```xml
{  
  "name": "calico-k8s-network",  
  "cniVersion": "0.1.0",  
  "type": "calico",  
    <!--etcd的url-->
    "etcd_endpoints": "http://192.168.1.102:2379",  
    "logevel": "info",  
    "ipam": {  
        "type": "calico-ipam"  
   },  
    "kubernetes": {  
        <!--api-server的url-->
        "k8s_api_root": "http://192.168.1.102:8080"  
    }  
}  
```

## 8. 练习

尝试一下kubernetes官方提供的入门教学：playground，了解如何创建pod，deployment，以及查看他们的信息，深入理解他们的关系。

```bash
## 查看当前系统日志
$ journalctl -f
## 查看kubernetes的一些版本信息
[root@server02 services]# kubectl version
Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T21:07:38Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T20:55:30Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
## 获取 get 命令的使用方法
$ kubectl get --help
## 获取所有的节点信息
[root@server02 services]# kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
192.168.1.101   Ready     <none>    3d        v1.9.0
192.168.1.103   Ready     <none>    3d        v1.9.0
$ kubectl run '部署名称' --image='容器名称' --port=8080   --创建一个部署<默认一个pod>
## 测试创建一个部署
$ kubectl run kubernetes-bootcamp --image=jocatalin/kubernetes-bootcamp:v1 --port=8080 
## 获取所有的部署，数据如下：
$ kubectl get deployments  
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1         1         1            0           14s
## DESIRED--期望pod数  CURRENT--当前pod数  UP-TO-DATE--最新pod数 AVAILABLE--可用pod数
## 获取所有的pod信息
[root@server02 services]# kubectl get pods
NAME                                   READY     STATUS    RESTARTS   AGE
kubernetes-bootcamp-5cbb65dd94-4sv4d   1/1       Running   1          3d
nginx                                  1/1       Running   1          3d
nginx-deployment-6c54bd5869-8hmms      1/1       Running   1          3d
nginx-deployment-6c54bd5869-sd26p      1/1       Running   1          3d
nginx-deployment-6c54bd5869-xwmm4      1/1       Running   1          3d
## 获取所有的pod显示更多的pod信息
[root@server02 services]# kubectl get pods -o wide
NAME                                   READY     STATUS    RESTARTS   AGE       IP              NODE
kubernetes-bootcamp-5cbb65dd94-4sv4d   1/1       Running   1          3d        172.20.40.195   192.168.1.103
nginx                                  1/1       Running   1          3d        172.20.188.3    192.168.1.101
nginx-deployment-6c54bd5869-8hmms      1/1       Running   1          3d        172.20.40.196   192.168.1.103
nginx-deployment-6c54bd5869-sd26p      1/1       Running   1          3d        172.20.40.197   192.168.1.103
nginx-deployment-6c54bd5869-xwmm4      1/1       Running   1          3d        172.20.188.2    192.168.1.101

## 获取'部署'里 labels中app=nginx的pod <service是根据labels里面app相同名称，来做负载均衡>
[root@server02 services]# kubectl get pods -l app=nginx
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-6c54bd5869-8hmms   1/1       Running   1          3d
nginx-deployment-6c54bd5869-sd26p   1/1       Running   1          3d
nginx-deployment-6c54bd5869-xwmm4   1/1       Running   1          3d
## 删除一个部署
$ kubectl delete deployments '部署名字'
[root@server02 services]# kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1         1         1            1           3d
nginx-deployment      3         3         3            3           3d
[root@server02 services]# kubectl delete deployments nginx-deployment
deployment "nginx-deployment" deleted
[root@server02 services]# kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1         1         1            1           3d

## 描述一个部署的详细信息
$ kubectl describe deployment '部署名称'
[root@server02 services]# kubectl describe deployment kubernetes-bootcamp
Name:                   kubernetes-bootcamp
Namespace:              default
CreationTimestamp:      Sat, 06 Jul 2019 07:35:17 +0000
Labels:                 run=kubernetes-bootcamp
Annotations:            deployment.kubernetes.io/revision=1
Selector:               run=kubernetes-bootcamp
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  run=kubernetes-bootcamp
  Containers:
   kubernetes-bootcamp:
    Image:        jocatalin/kubernetes-bootcamp:v2
    Port:         9090/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   kubernetes-bootcamp-5cbb65dd94 (1/1 replicas created)
Events:          <none>

## 描述一个 pod 的详细信息<名字可以使用：kubectl get pods 命令查看>
$ kubectl describe pods 'pod名称'
[root@server02 services]# kubectl get pods
NAME                                   READY     STATUS    RESTARTS   AGE
kubernetes-bootcamp-5cbb65dd94-4sv4d   1/1       Running   1          3d
nginx                                  1/1       Running   1          3d
[root@server02 services]# kubectl describe pods nginx
Name:         nginx
Namespace:    default
Node:         192.168.1.101/192.168.1.101
Start Time:   Sat, 06 Jul 2019 08:32:53 +0000
Labels:       <none>
Annotations:  <none>
Status:       Running
IP:           172.20.188.3
Containers:
  nginx:
    Container ID:   docker://eaffb766dba945aa883feae6845f93e2a9909efb088bd117bb07d7366e350e11
    Image:          nginx:1.7.9
    Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
    Port:           80/TCP
    State:          Running
      Started:      Sun, 07 Jul 2019 02:03:49 +0000
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Sat, 06 Jul 2019 08:33:08 +0000
      Finished:     Sat, 06 Jul 2019 10:18:53 +0000
    Ready:          True
    Restart Count:  1
    Environment:    <none>
    Mounts:         <none>
Conditions:
  Type           Status
  Initialized    True
  Ready          True
  PodScheduled   True
Volumes:         <none>
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     <none>
Events:          <none>

## 在当前机器上起一个 8001 的代理
$ kubectl proxy
## 重启一个窗口，执行一条命令来测试pod是否启动
$ curl http://localhost:8001/api/v1/proxy/namespaces/default/pods/pod名称/    --

## 为某个部署扩缩容数量
$ kubectl scale deployments '部署名称' --replicas=4
[root@server02 services]# kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1         1         1            1           3d
[root@server02 services]# kubectl scale deployments kubernetes-bootcamp --replicas=4
deployment "kubernetes-bootcamp" scaled
[root@server02 services]# kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   4         4         4            2           3d
[root@server02 services]#

## 修改某个部署的镜像
$ kubectl set image deployment '部署名称' '容器名称'='新的容器名称'
## 测试修改命令
$ kubectl set image deployment kubernetes-bootcamp  \
> kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
## 查看上面的是否修改成功
$ kubectl rollout status deployment '部署名称' 
## 回滚上一步对某个部署的操作  <比如上面我更新了部署的镜像>
$ kubectl rollout undo deployment '部署名称'

使用配置文件的方式创建 pod，需要创建配置文件，具体如下：
    nginx-pod.yaml -- 文件名，内容如下
    apiVersion: v1
    kind: Pod                   #类型
    metadata:                   #源数据
      name: nginx
    spec:                       #说明
      containers:               #容器
        - name: nginx           #容器名称
          image: nginx:1.7.9    #镜像
          ports:
            - containerPort: 80  #容器端口
## 使用配置文件创建 pod
$ kubectl create -f 'yaml文件名称'
## 查看刚刚创建的pod是否成功
$ kubectl get pods
$ kubectl proxy
$ curl http://127.0.0.1:8001/api/v1/proxy/namespaces/default/pods/nginx/

使用配置文件的方式创建 '部署'，需要创建配置文件，具体如下：
  nginx-deployment.yaml -- 文件名，内容如下
  apiVersion: apps/v1beta1
  kind: Deployment                 #类型
  metadata:                        #源数据
    name: nginx-deployment
  spec:
    replicas: 3                    #副本数<就是部署时有几个实列，启动几个服务>
    template:                      #对什么进行副本，根据什么区创建副本
      metadata: 
        labels:
          app: nginx
      spec:                        #说明
        containers:                #容器
          - name: nginx            #容器名称
            image: nginx:1.7.9     #镜像
            ports:
              - containerPort: 80  #容器端口
##  使用配置文件创建 '部署'                
$ kubectl create -f 'yaml文件名称'
[root@server02 services]# kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   4         4         4            4           3d
[root@server02 services]# kubectl create -f nginx-deployment.yaml
deployment "nginx-deployment" created
## 查看刚刚创建的 '部署' 是否成功
$ kubectl get deployments
[root@server02 services]# kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   4         4         4            4           3d
nginx-deployment      3         3         3            3           4s
##  获取'部署'里labels里面 app=nginx的pod  <service是根据labels 里面 app相同名称，来做负载均衡>
$ kubectl get pods -l app=nginx 
[root@server02 services]# kubectl get pods -l app=nginx
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-6c54bd5869-dtc6l   1/1       Running   0          40s
nginx-deployment-6c54bd5869-fl4hf   1/1       Running   0          40s
nginx-deployment-6c54bd5869-vdksg   1/1       Running   0          40s
## kubectl logs 'pod名称'，查看pod日志， -f：跟踪实时日志
[root@server02 services]# kubectl logs kubernetes-bootcamp-5cbb65dd94-4sv4d
Error from server: Get http://192.168.1.103:10250/containerLogs/default/kubernetes-bootcamp-5cbb65dd94-4sv4d/kubernetes-bootcamp: net/http: HTTP/1.x transport connection broken: malformed HTTP response "\x15\x03\x01\x00\x02\x02"
```

## 9. 为集群增加service功能 - kube-proxy（工作节点）
#### 9.1 简介
每台工作节点上都应该运行一个kube-proxy服务，它监听API server中service和endpoint的变化情况，并通过iptables等来为服务配置负载均衡，是让我们的服务在集群外可以被访问到的重要方式。
#### 9.2 部署
**通过系统服务方式部署：**
```bash
#确保工作目录存在
$ mkdir -p /var/lib/kube-proxy
#复制kube-proxy服务配置文件
$ cp ./target/worker-node/kube-proxy.service /lib/systemd/system/
#复制kube-proxy依赖的配置文件
$ cp ./target/worker-node/kube-proxy.kubeconfig /etc/kubernetes/

$ systemctl enable kube-proxy.service
$ service kube-proxy start
$ journalctl -f -u kube-proxy
```
#### 9.3 重点配置说明
**kube-proxy.service**
> [Unit]  
Description=Kubernetes Kube-Proxy Server
...  
[Service]  
\#工作目录  
WorkingDirectory=/var/lib/kube-proxy  
ExecStart=/home/michael/bin/kube-proxy \\  
\#监听地址  
  --bind-address=192.168.1.103 \\  
  \#依赖的配置文件，描述了kube-proxy如何访问api-server  
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\  
...

**kube-proxy.kubeconfig**
配置了kube-proxy如何访问api-server，内容与kubelet雷同，不再赘述。

```bash
问题描述：是kube-proxy组件没成功调iptables添加相关规则原因

server01 kube-proxy[11923]: I0706 09:39:07.211308   11923 proxier.go:1669] Closing local ports after iptables-restore failure
server01 kube-proxy[11923]: I0706 09:39:07.211316   11923 port.go:63] Closing local port "nodePort for default/nginx-deployment:" (:25594/tcp)
server01 kube-proxy[11923]: I0706 09:39:37.246611   11923 proxier.go:1754] Opened local port "nodePort for default/nginx-deployment:" (:25594/tcp)
server01 kube-proxy[11923]: E0706 09:39:37.248965   11923 proxier.go:1667] Failed to execute iptables-restore: exit status 1 (iptables-restore: invalid option -- '5'
Jul 06 09:39:37 server01 kube-proxy[11923]: Try `iptables-restore -h' for more information.
Jul 06 09:39:37 server01 kube-proxy[11923]: )
```

```bash
问题解决：
更换iptables的版本号降低到iptables-1.4.21-24.1.el7_5.x86
重启kube-proxy组件查看iptables规则有kube-proxy添加进来的规则

1、下载iptables-1.4.21-24.1.el7_5.x86_64.rpm
ftp://ftp.pbone.net/mirror/ftp.scientificlinux.org/linux/scientific/7.5/x86_64/updates/fastbugs/iptables-1.4.21-24.1.el7_5.x86_64.rpm
2、卸载本机的iptable
$ rpm -qa | grep iptables
 iptables-1.4.21-28.el7.x86_64
$ rpm -e iptables-1.4.21-28.el7.x86_64 --nodeps
```



#### 9.4 操练service
```bash
## 在装有apiServer节点执行查看当前服务
[root@server02 services]# kubectl get svc
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes            ClusterIP   10.68.0.1       <none>        443/TCP        3d
kubernetes-bootcamp   NodePort    10.68.140.17    <none>        80:27719/TCP   5h
nginx-deployment      NodePort    10.68.255.199   <none>        80:22226/TCP   3d
## 查看服务详细信息（服务名称可通过 kubectl get service 获取）
$ kubectl describe service '服务名字'
[root@server02 services]# kubectl describe service kubernetes
Name:              kubernetes
Namespace:         default
Labels:            component=apiserver
                   provider=kubernetes
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP:                10.68.0.1
Port:              https  443/TCP
TargetPort:        6443/TCP
Endpoints:         10.0.2.15:6443
Session Affinity:  ClientIP
Events:            <none>

    "暴露" 一个 "部署"   部署的名称      "暴露类型" "目标的端口（容器的端口）" "服务的端口"（内网服务的端口，和 CLUSTER-IP（看下面）绑定的端口，仅限内网使用）
       |       |         |           |              |                |
$ kubectl expose deploy nginx-deployment --type="NodePort" --target-port=80 --port=80     
#暴露（创建）服务（Service），随机生成端口

$ kubectl get deploy
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1         1         1            1           2h
nginx-deployment      3         3         3            3           1h
$ kubectl expose deploy nginx-deployment --type="NodePort" --target-port=80 --port=80
service "nginx-deployment" exposed
## 验证上面的命令是否创建service成功，显示数据如下
$ kubectl get services
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.68.0.1       <none>        443/TCP        1d
nginx-deployment   NodePort    10.68.193.138   <none>        80:38661/TCP   9s

第二条数据看 "PORT(S)"项，从80（内网服务的端口）映射到了38661（随机生成的）端口，可使用如下命令查看端口监听情况：
$ netstat -ntlp | grep 38661    
-- 如果当前机器起了 kube-proxy 是会有监听的，而这个端口实际是kube-proxy在节点上启动的一个端口，
-- 可通过这个这台机器的ip+这个端口（38661）来访问我们 的服务（前提是该机器跑有 kube-proxy）
-- 到服务（service）里面的容器中使用 CLUSTER-IP（看上面）+ （内网服务的端口）是可以访问的
-- 在节点内部使用pod的ip+容器的端口是可以访问的，或者使用   CLUSTER-IP（看上面）+ （内网服务的端口）也是可以访问的

## kube-proxy的工作节点查看监听进程
$ netstat -ntlp | grep 38661

使用配置文件的方式创建 service,创建文件 nginx-service.yaml 内容如下：
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports: 
  - port: 80            #服务的端口（内网服务的端口，和 CLUSTER-IP（看下面）绑定的端口，仅限内网使用
    targetPort: 80      #目标的端口（容器的端口
    nodePort: 20000     #node节点绑定的端口，就是机器对外提供服务的端口
  selector:
    app: nginx          #选择给谁提供服务，这里选的是 deployments（部署） 里 labels 下 app=nginx 的部署。（可以看上面我们创建了一个 deployments（部署）里面有 app:nginx）
  type: NodePort

[root@server02 services]# kubectl get svc
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP   10.68.0.1       <none>        443/TCP          3d
kubernetes-bootcamp   NodePort    10.68.140.17    <none>        80:27719/TCP     5h
nginx-deployment      NodePort    10.68.255.199   <none>        80:22226/TCP     3d
nginx-service         NodePort    10.68.180.129   <none>        8080:20000/TCP   4h
## 创建 service
[root@server02 services]# kubectl delete -f nginx-service.yml
service "nginx-service" deleted
## 看看创建的 service 是否成功
[root@server02 services]# kubectl get svc
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes            ClusterIP   10.68.0.1       <none>        443/TCP        3d
kubernetes-bootcamp   NodePort    10.68.140.17    <none>        80:27719/TCP   5h
nginx-deployment      NodePort    10.68.255.199   <none>        80:22226/TCP   3d
```

## 10. 为集群增加dns功能 - kube-dns（app<和普通应用差不不多>，MASTER）
#### 10.1 简介
kube-dns为Kubernetes集群提供命名服务，主要用来解析集群服务名和Pod的hostname。目的是让pod可以通过名字访问到集群内服务。它通过添加A记录的方式实现名字和service的解析。普通的service会解析到service-ip。headless service会解析到pod列表。
#### 10.2 部署
**通过kubernetes应用的方式部署**
kube-dns.yaml文件基本与官方一致（除了镜像名不同外）。
里面配置了多个组件，之间使用”---“分隔

```bash
## 到kubernetes-starter目录执行命令
## 这个是官方提供创建 dns 服务的配置（里面有注释，可以看看）
$ kubectl create -f ./target/services/kube-dns.yaml
configmap "kube-dns" created
serviceaccount "kube-dns" created
service "kube-dns" created
deployment "kube-dns" created
## 查看kube-dns 服务是否创建成功，-n 是制定命名空间，kube-system 是 kubernetes 系统内部的命名空间
$ kubectl -n kube-system get services
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.68.0.2    <none>        53/UDP,53/TCP   2m
## 查看kube-dns 部署是否创建成功
$ kubectl -n kube-system get deployments
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-dns   1         1         1            1           2m
## 查看kube-dns 的 pod是否运行
$ kubectl -n kube-system get pods -o wide
NAME                       READY     STATUS    RESTARTS   AGE       IP             NODE
kube-dns-77ddf98fc-jzpts   3/3       Running   0          3m        172.20.188.4   192.168.1.101
$ kubectl -n kube-system get pods
NAME                       READY     STATUS    RESTARTS   AGE
kube-dns-77ddf98fc-jzpts   3/3       Running   0          5m
## 到 dns 运行的pod上执行，查看运行了那些容器
$ docker ps | grep dns                                        -- 
     一般会运行如下几个容器：
      k8s-dns-sidecar：用于监控其他几个容器的健康状态
      k8s-dns-dnsmasq：用于 dns 缓存，来提升效率
      k8s-dns-kube-dns：真正提供 dns 服务的容器
      pause-amd64：pod 容器
## 删除 kube-dns 服务
$ kubectl delete -f /home/kubernetes/kubernetes-starter/target/services/kube-dns.yaml
configmap "kube-dns" deleted
serviceaccount "kube-dns" deleted
service "kube-dns" deleted
deployment "kube-dns" deleted
## 查看service
[root@server02 kubernetes-starter]# kubectl get svc
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP   10.68.0.1       <none>        443/TCP          3d
kubernetes-bootcamp   NodePort    10.68.140.17    <none>        80:27719/TCP     1h
nginx-deployment      NodePort    10.68.255.199   <none>        80:22226/TCP     2d
nginx-service         NodePort    10.68.180.129   <none>        8080:20000/TCP   11m
```
#### 10.3 重点配置说明
请直接参考配置文件中的注释。

#### 10.4 通过dns访问服务，到主节点上执行如下操作
```bash
## 查看所有的 services
$ kubectl get services  
## 找一个装有 curl 的容器
$ kubectl get pods -o wide  
## 进入容器内部执行如下命令：
$ docker exec -it '容器的ID（docker ps 查看）'
## 验证使用名称是否可以访问服务
$ curl '服务的名称（kubectl get services）':内网端口
## 查看当前容器的 dns 配置
$ cat /etc/resolv.conf
## 删除 dns 服务（不要做这一步）
$ kubectl delete -f ./target/services/kube-dns.yaml 
```
