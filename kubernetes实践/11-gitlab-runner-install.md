# GitLab-Runner安装和使用教程（Docker版）

## 一、准备环境

### 1. 官网安装地址

官方安装地址：https://docs.gitlab.com/runner/install/

### 2. Install Docker First

```bash
$ curl -sSL https://get.docker.com/ | sh
```

## 二、下载镜像

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

## 三、运行GitLab-Runner容器

### 1. 关闭selinux

```bash
[root@server05 gitlab-runner]# getenforce
Enforcing
[root@server05 gitlab-runner]# setenforce 0
[root@server05 gitlab-runner]# getenforce
Permissive
[root@server05 gitlab-runner]# vim /etc/sysconfig/selinux
SELINUX=disabled
```

### 2. 运行start.sh 启动gitlab

Install as a Docker Service:https://docs.gitlab.com/runner/install/docker.html

```bash
$ tee start.sh <<-'EOF'
#!/bin/bash
GITLAB_DIR=`pwd`
docker pull gitlab/gitlab-runner:latest
docker stop gitlab-runner
docker rm gitlab-runner
docker run -d --name gitlab-runner --restart always \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v ${GITLAB_DIR}/config:/etc/gitlab-runner \
       gitlab/gitlab-runner:latest
docker exec -it gitlab-runner touch /etc/gitlab-runner/config.toml
EOF
```

### 3. 运行start.sh 启动gitlab-runner

```bash
$ sh start.sh
latest: Pulling from gitlab/gitlab-runner
Digest: sha256:20218167297797ac38aaa7b7bcae78997981fecc685ba0208813cec1e0cc9047
Status: Image is up to date for gitlab/gitlab-runner:latest
gitlab-runner
gitlab-runner
542cd96e5f2d1a1ddef099d01f6fe29539e6d81eacc2ba29e193e0d466e889f2
$ docker logs -f gitlab-runner
Runtime platform                                    arch=amd64 os=linux pid=6 revision=d0b76032 version=12.0.2
Starting multi-runner from /etc/gitlab-runner/config.toml ...  builds=0
Running in system-mode.

Configuration loaded                                builds=0
listen_address not defined, metrics & debug endpoints disabled  builds=0
[session_server].listen_address not defined, session endpoints disabled  builds=0
Configuration loaded                                builds=0
```

### 4. 注册runner

- Go to Settings > CICD to obtain the token

  <img src="images/CICD-Setting.png" width="100%" height="100%"/>

- Register the Runner(Docker)

  Register the Runner：https://docs.gitlab.com/runner/register/

  <img src="images/CICD-Runner.png" width="100%" height="100%"/>

  ```bash
  ## 运行 register 命令(docker)
  $ docker exec -it gitlab-runner gitlab-runner register
  Runtime platform                   arch=amd64 os=linux pid=39 revision=d0b76032 version=12.0.2
  Running in system-mode.
  ## 输入你的GitLab实例URL:
  Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
  http://192.168.1.105
  ## 输入你获得的令牌，注册Runner：
  Please enter the gitlab-ci token for this runner:
  gmihm2iLjPE-ojQpbr37
  ## 输入Runner的描述，你可以稍后在GitLab的UI中更改：
  Please enter the gitlab-ci description for this runner:
  [542cd96e5f2d]: java8-runner
  ## 输入与Runner关联的标签，你可以稍后在GitLab的UI中进行更改：
  Please enter the gitlab-ci tags for this runner (comma separated):
  java8-runner
  Registering runner... succeeded                     runner=gmihm2iL
  ## 输入 Runner executor:
  Please enter the executor: shell, ssh, virtualbox, docker+machine, docker-ssh+machine, docker, docker-ssh, parallels, kubernetes:
  docker
  ## If you chose Docker as your executor, you’ll be asked for the default image to be used for 
  ## projects that do not define one in .gitlab-ci.yml:
  Please enter the default Docker image (e.g. ruby:2.6):
  java:8
  Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
  ```

  ```bash
  $ docker exec -it gitlab-runner gitlab-runner register
  Runtime platform                     arch=amd64 os=linux pid=51 revision=d0b76032 version=12.0.2
  Running in system-mode.
  
  Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
  http://192.168.1.105
  Please enter the gitlab-ci token for this runner:
  gmihm2iLjPE-ojQpbr37 ## token
  Please enter the gitlab-ci description for this runner:
  [542cd96e5f2d]: shell-runner
  Please enter the gitlab-ci tags for this runner (comma separated):
  shell-runner
  Registering runner... succeeded                     runner=gmihm2iL
  Please enter the executor: docker+machine, kubernetes, docker-ssh, ssh, shell, virtualbox, docker-ssh+machine, docker, parallels:
  shell
  Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
  ```

### 5.查看已经注册的Runner的信息

- 查看Runner的基本信息

  <img src="images/runner-1.png" width="100%" height="100%"/>

  <img src="images/runner-2.png" width="100%" height="100%"/>

- 修改Runner的基本信息

  <img src="images/runner-3.png" width="100%" height="100%"/>

## 四、GitLab-Runner试用

### 1. 推荐的官方连接

GitLab Runner Commands:https://docs.gitlab.com/runner/commands/README.html

### 2. Creating and Tweaking GitLab CI/CD for GitLab Pages

```yaml
image: ruby:2.3

cache:
  paths:
  - vendor/

before_script:
  - bundle install --path vendor

pages:
  stage: deploy
  script:
  - bundle exec jekyll build -d public
  artifacts:
    paths:
    - public
  only:
  - master

test:
  stage: test
  script:
  - bundle exec jekyll build -d test
  artifacts:
    paths:
    - test
  except:
  - master
```

- ### Creating a simple `.gitlab-ci.yml` file

```yaml
image: "ruby:2.5"

before_script:
  - apt-get update -qq && apt-get install -y -qq sqlite3 libsqlite3-dev nodejs
  - ruby -v
  - which ruby
  - gem install bundler --no-document
  - bundle install --jobs $(nproc)  "${FLAGS[@]}"

rspec:
  script:
    - bundle exec rspec

rubocop:
  script:
    - bundle exec rubocop
```

- ### Push `.gitlab-ci.yml` to GitLab

```bash
git add .gitlab-ci.yml
git commit -m "Add .gitlab-ci.yml"
git push origin master
```

### 3. Java with Spring Boot

- [Deploy a Spring Boot application to Cloud Foundry with GitLab CI/CD](https://docs.gitlab.com/ee/ci/examples/deploy_spring_boot_to_cloud_foundry/index.html).

  链接：https://docs.gitlab.com/ee/ci/examples/deploy_spring_boot_to_cloud_foundry/index.html

- 新建一个Spring Boot项目

- Configure the deployment to Cloud Foundry

  新建一个`manifest.yml`

  ```yaml
  applications:
  - name: gitlab-hello-world
    random-route: true
    memory: 1G
    path: target/demo-0.0.1-SNAPSHOT.jar
  ```

  

- Configure GitLab CI/CD to deploy your application

  - 新建一个`.gitlab-ci.yml`
  - 将文件放在项目的根目录下面

  

  ```yaml
  image: java:8
  
  stages:
    - build
    - deploy
  
  before_script:
    - chmod +x mvnw
  
  build:
    stage: build
    script: ./mvnw package
    artifacts:
      paths:
        - target/demo-0.0.1-SNAPSHOT.jar
  
  production:
    stage: deploy
    script:
    - curl --location "https://cli.run.pivotal.io/stable?release=linux64-binary&source=github" | tar zx
    - ./cf login -u $CF_USERNAME -p $CF_PASSWORD -a api.run.pivotal.io
    - ./cf push
    only:
    - master
  ```

  

### 4. Java with Maven

- [How to deploy Maven projects to Artifactory with GitLab CI/CD](https://docs.gitlab.com/ee/ci/examples/artifactory_and_gitlab/index.html).

  官方链接：https://docs.gitlab.com/ee/ci/examples/artifactory_and_gitlab/index.html

- 新建一个Maven项目

  - Log in to your GitLab account

  - Create a new project by selecting **Import project from ➔ Repo by URL** 

  - Add the following URL:

    ```
    https://gitlab.com/gitlab-examples/maven/simple-maven-dep.git
    ```

  - Click **Create project** 
    -  `pom.xml`: project object model (POM) configuration file
    -  `src/main/java/com/example/dep/Dep.java`: source of our application

- Configure the Artifactory deployment

- Configure GitLab CI/CD for `simple-maven-dep`

  -  **MAVEN_REPO_URL**: `http://artifactory.example.com:8081/artifactory` (your Artifactory URL)
  -  **MAVEN_REPO_USER**: `gitlab` (your Artifactory username)
  -  **MAVEN_REPO_PASS**: `AKCp2WXr3G61Xjz1PLmYa3arm3yfBozPxSta4taP3SeNu2HPXYa7FhNYosnndFNNgoEds8BCS` (your Artifactory Encrypted Password)

  ```yaml
  image: maven:latest
  
  variables:
    MAVEN_CLI_OPTS: "-s .m2/settings.xml --batch-mode"
    MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  
  cache:
    paths:
      - .m2/repository/
      - target/
  
  build:
    stage: build
    script:
      - mvn $MAVEN_CLI_OPTS compile
  
  test:
    stage: test
    script:
      - mvn $MAVEN_CLI_OPTS test
  
  deploy:
    stage: deploy
    script:
      - mvn $MAVEN_CLI_OPTS deploy
    only:
      - master
  ```

### 5.Hello Docker for GitLab-Runner

#### 5-1.在gitlab上新建spring-boot项目：hello-docker

链接：ssh://git@gitlab.gusuchen.com:2222/gusuchen/hello-docker.git

#### 5-2.

#### 5-3.

#### 5-4.

#### 5-5.

#### 5-6.

### 6.MicroService for GitLab-Runner

#### 6-1.

#### 6-2

#### 6-3

#### 6-4

#### 6-5

### 7.Pipeline Demo

#### 7-1.创建一个空白的项目：hello-world

#### 7-2.在该项目中注册一个Runner​：shell

<img src="images/runner-shell.png" width="100%" height="100%"/>

```bash
$ docker exec -it gitlab-runner gitlab-runner register
Runtime platform                     arch=amd64 os=linux pid=51 revision=d0b76032 version=12.0.2
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://192.168.1.105
Please enter the gitlab-ci token for this runner:
gmihm2iLjPE-ojQpbr37 ## token
Please enter the gitlab-ci description for this runner:
[542cd96e5f2d]: shell-runner
Please enter the gitlab-ci tags for this runner (comma separated):
shell-runner
Registering runner... succeeded                     runner=gmihm2iL
Please enter the executor: docker+machine, kubernetes, docker-ssh, ssh, shell, virtualbox, docker-ssh+machine, docker, parallels:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

#### 7-3.创建 `.gitlab-ci.yml` 文件

```yaml
stages:
  - build
  - test
  - deploy
job1:
  stage: build
  script:
    - echo "I am job1"
    - echo "I am in build stage"
  tags:
    - shell-runner
job2:
  stage: test
  script:
    - echo "I am job2"
    - echo "I am in test stage"
  tags:
    - shell-runner
job3:
  stage: deploy
  script:
    - echo "I am job3"
    - echo "I am in deploy stage"
  tags:
    - shell-runner
```

#### 7-4.提交`.gitlab-ci.yml`文件，触发自动构建

<img src="images/pepline-1.png" />

<img src="images/pepline-2.png" />

<img src="images/pepline-3.png" />

## 五、问题记录

### 1.Could not resolve host

```
Reinitialized existing Git repository in /builds/gusuchen/hello-docker/.git/
fatal: unable to access 'http://gitlab-ci-token:xxxxxxxxxxxxxxxxxxxx@gitlab.gusuchen.com/gusuchen/hello-docker.git/': Could not resolve host: gitlab.gusuchen.com
ERROR: Job failed: exit code 1
```

官网解决：https://forum.gitlab.com/t/unable-to-access-in-runner/9400/2

```yaml
在/etc/gitlab-runner/config.toml中追加:extra_hosts = ["gitlab.gusuchen.com:192.168.1.105"]
[[runners]]
  name = "mvn3_jdk8-runner"
  url = "http://192.168.1.105"
  token = "07747d318645dcdb4ab81f63995194"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "maven:3-jdk-8"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
    extra_hosts = ["gitlab.gusuchen.com:192.168.1.105"]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```

### 2.build和test都会重新拉取镜像，下载依赖的jar包，耗时耗资源

```markdown
原因：这是因为docker image每次构建都是在独立的container里, maven的.m2文件并不会被多次构建公用
```

```markdown
解决办法：修改gitlab-runner的配置，将maven的.m2目录加到volumes中，并增加镜像拉取规则（默认是从远程拉取镜像，这里修改为优先获取本地镜像，不存在时才去远程拉取镜像）。
```

```bash
$ vim config.toml
[[runners]]
  name = "mvn3_jdk8-runner"
  url = "http://192.168.1.105"
  token = "07747d318645dcdb4ab81f63995194"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "maven:3-jdk-8"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/home/vagrant/labs/.m2:/root/.m2"]
    pull_policy = "if-not-present"
    shm_size = 0
    extra_hosts = ["gitlab.gusuchen.com:192.168.1.105"]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]

##在 /home/vagrant/labs/.m2下面新建setting.xml文件
$ cd /home/vagrant/lab/.m2
$ tee setting.xml <<-'EOF'
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository>/usr/share/maven/ref/repository</localRepository>

  <mirrors>
    <mirror>
      <id>nexus-aliyun</id>
      <mirrorOf>central</mirrorOf>
      <name>Nexus aliyun</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
  </mirrors>
  
</settings>
EOF
```

之后每次下载后的jar包都会被缓存在本地maven的repository中，不需要再重新下载。

### 3.docker：command not found

链接:https://docs.gitlab.com/ee/ci/docker/using_docker_build.html

```bash
Skipping Git submodules setup
$ echo "Deploy Start"
Deploy Start
$ sudo docker build -t gusuchen163/hello-docker:latest .
bash: line 62: sudo: command not found
ERROR: Job failed: exit status
```

```bash
解决：
## During GitLab Runner installation select shell as method of executing job scripts or use command:
$ sudo gitlab-runner register -n \
   --url https://gitlab.com/ \
   --registration-token REGISTRATION_TOKEN \
   --executor shell \
   --description "My Runner"
## Add gitlab-runner user to docker group:
$ sudo usermod -aG docker gitlab-runner
## Verify that gitlab-runner has access to Docker:
$ sudo -u gitlab-runner -H docker info
 before_script:
   - docker info

 build_image:
   script:
     - docker build -t my-docker-image .
     - docker run my-docker-image /script/to/run/tests

```

在runner上挂载宿主机的docker命令

```
[[runners]]
  name = "mvn3_jdk8-runner"
  url = "http://192.168.1.105"
  token = "07747d318645dcdb4ab81f63995194"
  executor = "docker"
  clone_url = "http://192.168.1.105"
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "maven:3-jdk-8"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/home/vagrant/labs/.m2:/root/.m2", "/var/run/docker.sock:/var/run/docker.sock", "/bin/docker:/bin/docker"]
    extra_hosts = ["gitlab.gusuchen.com:192.168.1.105"]
    pull_policy = "if-not-present"
    shm_size = 0
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```

4.