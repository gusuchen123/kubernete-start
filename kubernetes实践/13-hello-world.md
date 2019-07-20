## Win10 docker 部署redis:latest

```shell
$ docker pull redis
$ docker run -d --name redis \
> -p 6379:6379 \
> -v e:/docker/redis-data:/data 
> redis:latest
```

## win10 docker 部署mysql:5.7.26

```shell
$ docker pull mysql:5.7.26

$ docker run --name mysql \
> --env MYSQL_ROOT_PASSWORD=root \
> --env MYSQL_DATABASE=db_user \
> -v e:/docker/mysql-data:/var/lib/mysql \
> -p 3306:3306 \
# > --network demo \
# > --mount type=volume,source=mysql-data,destination=/var/lib/mysql \
> mysql:5.7.26
```

## win10 docker 部署zookeeper:latest

```shell
$ docker pull zookeeper

$ docker run --name zookeeper \
> --restart always \
> -p 2081:2081 \
> zookeeper
```

## docker-compose 部署hello-world + mysql

- spring-boot应用的Dockerfile

  ```dockerfile
  FROM java:8-jre
  MAINTAINER gusuchen gusuchen@thunisoft.com
  
  COPY /target/hello-world-0.0.1-SNAPSHOT.jar /hello-world.jar
  ENTRYPOINT ["java", "-jar", "/hello-world.jar"]
  ```

  

- docker-compose.yml

  ```yaml
  version: '3'
  
  services:
    db:
      image: mysql:5.7.26
      container_name: mysql-5.7
      restart: always
      environment:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: db_user
      ports:
        - 3306:3306
      volumes:
        - e:/docker/mysql-data:/var/lib/mysql
  
    hello-world-service:
      image: gusuchen163/hello-world:v2
      container_name: hello-world-service
      restart: always
      command:
        - "--mysql.address=172.23.22.137"
      ports:
        - 8080:8080
  ```

  

- build.sh

  ```bash
  #!/usr/bin/env bash
  mvn clean package -Dmaven.test.skip=true
  
  docker build -t gusuchen163/hello-world:v2 .
  
  docker run -it -d --name hello-world -p 8080:8080 gusuchen163/hello-world:v2 --mysql.address=172.23.22.137
  
  docker-compose up -d
  ```