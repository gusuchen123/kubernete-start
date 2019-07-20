# 容器编排Docker Swarm

## Docker Swarm介绍



## 创建一个3节点的swarm集群

swarm-manger、swarm-worker1、swarm-worker2

在主机上新创建的节点默认是manager节点

```shell
[vagrant@swarm-manager ~]$ docker swarm init --advertise-addr=192.168.205.20
Swarm initialized: current node (j2i9s03rzq7qu802898y1jq8w) is now a manager.
To add a worker to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-1wxyemnp89skmo5e4wrfw5859g0rhxfbby7udwpg4d8vmfmpa1-1pbtfwouvom7fjcq04a4cmqz0 192.168.205.20:2377
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

[vagrant@swarm-worker1 ~]$ docker swarm join --token SWMTKN-1-1wxyemnp89skmo5e4wrfw5859g0rhxfbby7udwpg4d8vmfmpa1-1pbtfwouvom7fjcq04a4cmqz0 192.168
This node joined a swarm as a worker.

[vagrant@swarm-worker2 ~]$ docker swarm join --token SWMTKN-1-1wxyemnp89skmo5e4wrfw5859g0rhxfbby7udwpg4d8vmfmpa1-1pbtfwouvom7fjcq04a4cmqz0 192.168
This node joined a swarm as a worker.

[vagrant@swarm-manager ~]$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
j2i9s03rzq7qu802898y1jq8w *   swarm-manager       Ready               Active              Leader              18.09.6
uigqallnbiii14syge75lwq46     swarm-worker1       Ready               Active                                  18.09.6
gmgr8kqvgoz87ckmz5iy3sac1     swarm-worker2       Ready               Active                                  18.09.6
```

## Service的创建维护和水平扩展

### Service 创建

```shell
[vagrant@swarm-manager ~]$ docker service create --name demo busybox sh -c 'while true; do sleep 3600; done'
ta4jzaww7othhn1a8ldhyvl4v
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
[vagrant@swarm-manager ~]$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
ta4jzaww7oth        demo                replicated          1/1                 busybox:latest
[vagrant@swarm-manager ~]$ docker service ps demo
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
nxdyk919lnl3        demo.1              busybox:latest      swarm-manager       Running             Running 2 minutes ago
[vagrant@swarm-manager ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
05ee61c0529f        busybox:latest      "sh -c 'while true; …"   2 minutes ago       Up 2 minutes                            demo.1.nxdyk919lnl33gob99fyf2uth
```

### 水平扩展 docker service scale

```shell
[vagrant@swarm-manager ~]$ docker service scale
"docker service scale" requires at least 1 argument.
See 'docker service scale --help'.
Usage:  docker service scale SERVICE=REPLICAS [SERVICE=REPLICAS...]
Scale one or multiple replicated services

[vagrant@swarm-manager ~]$ docker service scale demo=5
demo scaled to 5
overall progress: 5 out of 5 tasks
1/5: running   [==================================================>]
2/5: running   [==================================================>]
3/5: running   [==================================================>]
4/5: running   [==================================================>]
5/5: running   [==================================================>]
verify: Service converged
[vagrant@swarm-manager ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
3ca41bbcec61        busybox:latest      "sh -c 'while true; …"   21 seconds ago      Up 21 seconds                           demo.5.ltcvq416i1rs5760u4rddn500
05ee61c0529f        busybox:latest      "sh -c 'while true; …"   4 minutes ago       Up 4 minutes                            demo.1.nxdyk919lnl33gob99fyf2uth
[vagrant@swarm-manager ~]$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
ta4jzaww7oth        demo                replicated          5/5                 busybox:latest
[vagrant@swarm-manager ~]$ docker service ps demo
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
nxdyk919lnl3        demo.1              busybox:latest      swarm-manager       Running           Running 7 minutes ago
kjoosq8gihch        demo.2              busybox:latest      swarm-worker1       Running           Running 2 minutes ago
so4xwwdjuxcx        demo.3              busybox:latest      swarm-worker2       Running           Running 2 minutes ago
wlrnxywmj9qz        demo.4              busybox:latest      swarm-worker2       Running           Running 2 minutes ago
ltcvq416i1rs        demo.5              busybox:latest      swarm-manager       Running           Running 3 minutes ago
```

### 在swarm集群中模拟故障，删除一个docker容器服务

模拟故障删除一个服务后，集群的机制会自动创建一个新的容器服务

#### 模拟故障前集群中的docker服务

```shell
[vagrant@swarm-manager ~]$ docker service ps demo
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
nxdyk919lnl3        demo.1              busybox:latest      swarm-manager       Running           Running 7 minutes ago
kjoosq8gihch        demo.2              busybox:latest      swarm-worker1       Running           Running 2 minutes ago
so4xwwdjuxcx        demo.3              busybox:latest      swarm-worker2       Running           Running 2 minutes ago
wlrnxywmj9qz        demo.4              busybox:latest      swarm-worker2       Running           Running 2 minutes ago
ltcvq416i1rs        demo.5              busybox:latest      swarm-manager       Running           Running 3 minutes ago
```

#### 模拟故障

```shell
[vagrant@swarm-worker2 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
850e5de9eb9c        busybox:latest      "sh -c 'while true; …"   5 minutes ago       Up 5 minutes                            demo.3.so4xwwdjuxcxxndvwen6wedri
c006b0253e02        busybox:latest      "sh -c 'while true; …"   5 minutes ago       Up 5 minutes                            demo.4.wlrnxywmj9qzb9qiv2qyzex1u
[vagrant@swarm-worker2 ~]$ docker rm -f 850e5de9eb9c
850e5de9eb9c
```

#### 故障之后swarm集群的状况

```shell
[vagrant@swarm-worker2 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
a9e387ab77c1        busybox:latest      "sh -c 'while true; …"   45 seconds ago      Up 40 seconds                           demo.3.mjugd3vuonx8kfjekknyv9ei1
c006b0253e02        busybox:latest      "sh -c 'while true; …"   6 minutes ago       Up 6 minutes                            demo.4.wlrnxywmj9qzb9
[vagrant@swarm-manager ~]$ docker service ps demo
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR                         PORTS
nxdyk919lnl3        demo.1              busybox:latest      swarm-manager       Running             Running 10 minutes ago                        
kjoosq8gihch        demo.2              busybox:latest      swarm-worker1       Running             Running 5 minutes ago                         
mjugd3vuonx8        demo.3              busybox:latest      swarm-worker2       Ready               Ready 4 seconds ago                           
so4xwwdjuxcx         \_ demo.3          busybox:latest      swarm-worker2       Shutdown            Failed 4 seconds ago     "task: non-zero exit (137)"
wlrnxywmj9qz        demo.4              busybox:latest      swarm-worker2       Running             Running 5 minutes ago                         
ltcvq416i1rs        demo.5              busybox:latest      swarm-manager       Running             Running 6 minutes ago                         
[vagrant@swarm-manager ~]$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
ta4jzaww7oth        demo                replicated          5/5                 busybox:latest 
```



## 在swarm集群里创建overlay网络

```shell
[vagrant@swarm-manager example-vote-app]$ docker network create -d overlay demo
nbwa8liw090qp365rrbg2kpts
[vagrant@swarm-manager example-vote-app]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
68cb9dc63da7        bridge              bridge              local
nbwa8liw090q        demo                overlay             swarm
b66e1ede818d        docker_gwbridge     bridge              local
51afa9ab8d7b        host                host                local
uagqz6eza1pz        ingress             overlay             swarm
913a5b21ad05        none 
```

## 在swarm集群里通过service部署wordpress

#### 使用 docker service 部署 mysql

```shell
[vagrant@swarm-manager ~]$ docker service create --name mysql \
> --env MYSQL_ROOT_PASSWORD=root \
> --env MYSQL_DATABASE=wordpress \
> --network demo \
> --mount type=volume,source=mysql-data,destination=/var/lib/mysql \
> mysql
image mysql:latest could not be accessed on a registry to record
its digest. Each node will access mysql:latest independently,
possibly leading to different nodes running different
versions of the image.

hjt895aunh9wlar1joq6zbtz5
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged

[vagrant@swarm-manager ~]$ docker service ps mysql
ID     NAME      IMAGE     NODE    DESIRED STATE     CURRENT STATE     ERROR        PORTS
i1eo6ndmn0qk  mysql.1  mysql:latest  swarm-manager  Running  Running 8 minutes ago
[vagrant@swarm-manager ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                 NAMES
8bf543926cc2        mysql:latest        "docker-entrypoint.s…"   10 minutes ago      Up 10 minutes       3306/tcp, 33060/tcp   mysql.1.i1eo6ndmn0qks4nlw1vj9xofi
```

#### 使用docker service 部署 wordpress

```shell
[vagrant@swarm-manager ~]$ docker service create --name wordpress \
> -p 80:80 \
> --env WORDPRESS_DB_PASSWORD=root \
> --env WORDPRESS_DB_HOST=mysql \
> --network demo \
> wordpress
sn2xiybtgv9php8rde3z3ju4j
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
[vagrant@swarm-manager ~]$ docker service ps wordpress
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
2y1e8aeofy9l        wordpress.1         wordpress:latest    swarm-worker1       Running             Running 54 seconds ago
[vagrant@swarm-worker1 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
0890543ac046        wordpress:latest    "docker-entrypoint.s…"   8 minutes ago       Up 7 minutes        80/tcp              wordpress.1.2y1e8aeofy9lyrm9h1e9gxssp
```

#### 在swarm-worker1节点的网络状况

会发现有 `nbwa8liw090q        demo                overlay             swarm`

```shell
[vagrant@swarm-worker1 ~]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
81e1842b0ba6        bridge              bridge              local
nbwa8liw090q        demo                overlay             swarm
9cfd0c6f0f64        docker_gwbridge     bridge              local
46c7f3a5cd22        host                host                local
uagqz6eza1pz        ingress             overlay             swarm
b152bf42cb82        none                null                local
```

```
docker service create --name whoami -p 8000:8000 --network demo -d jwilder/whoami
docker service ls
docker service ps whoami
curl 127.0.0.1:8000
docker service create --name client -d --network demo busybox sh -c 'while true; do sleep 3600; done'
docker service ls
docker service ps client


worker1节点
docker exec -it dsaj /bin/sh
ping whoami

docker service scale whoami=2

nslookup www.imooc.com
nslookup whoami
nslookup tasks.whoami
```