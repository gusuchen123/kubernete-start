# Vagrant 使用笔记

### 1、通过`ur`l链接远程下载一个Vagrant Box

但是下载速度会很慢，建议先进行下载到本地

```shell
$ vagrant box add {title} {url}
$ vagrant init {tile}
$ vagrant up
```

### 2、下载Vagrant Box到本地

```shell
## 将本地的CentOS-7-x86_64-Vagrant-1902_01.VirtualBox.box添加到Vagrant Box；
## 名称是：CentOS-7-x86_64-Vagrant-1902_01.VirtualBox.box
$ vagrant box add CentOS-7-1902_01 CentOS-7-x86_64-Vagrant-1902_01.VirtualBox.box
$ vagrant box list              ## 查看Vagrant Box中所有的box
$ mkdir centos-7-1902_01
$ cd centos-7-1902_01
$ vagrant init CentOS-7-1902_01 ## 初始化生成Vagrantfile文件
$ vagrant up                    ## 启动Vagrant box
```

### 3、Vagrant 常用命令

```shell
$ vagrant box add {title} {virtualbox名称|url} # 添加box
$ vagrant init <本地box名称> ## 初始化box
$ vagrant up          ## 启动Vagrant box
$ vagrant ssh         ## 连接虚拟机
$ vagrant box list    ## 列出Vagrant当前的box列表
$ vagrant box remove  ## 删除box
$ vagrant destory     ## 停止正在运行的虚拟机，并销毁所有的资源
$ vagrant halt        ## 关闭虚拟机
$ vagrant package     ## 把当前的运行的虚拟机环境进行打包为 box 文件
$ vagrant plugin      ## 安装卸载插件
$ vagrant reload      ## 重新启动虚拟机，且重新载入配置文件Vagrantfile
$ vagrant resume      ## 恢复被挂起的状态  
$ vagrant suspend     ## 挂起当前虚拟机
$ vagrant status      ## 获取当前虚拟机的状态
$ vagrant global-status ##  查看当前 vagrant 管理的所有 vm 信息
$ vagrant ssh-config  ## 输出用于ssh连接的信息
```

### 4、介绍`Vagrantfile`

将`Vagrantfile`和box共享给其他开发人员，既可以得到一摸一样的开发环境

```shell
## 执行 vagrant init <本地box名称>，会在当前目录生成Vagrantfile
$ vagrant init CentOS-7-1902_01
```

`Vagrantfile` 主要包括三个方面的配置，虚拟机的配置、SSH配置、Vagrant 的一些基础配置。

- box名称设置

  `  config.vm.box = "CentOS-7-x86_64-Vagrant-1905_01"`

- `VM` 相关配置

  用`modifyvm`这个命令让我们可以设定 `VM` 的名称和内存大小等等，这里说的名称指的是在 `VirtualBox` 中显示的名称，我们也可以在 `Vagrantfile` 中进行设定，举例如下：

  ```markdown
  config.vm.provider "virtualbox" do |v|
    v.customize ["modifyvm", :id, "--name", "centos-7", "--memory", "1024"]
  end
  ```

  ```markdown
  config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    vb.gui = true
  
    # Customize the amount of memory on the VM:
    vb.memory = "1024"
  end
  ```

- 网络设置

  - 主机模式： host-only ，所有的虚拟系统是可以互相通信的，但虚拟系统和真实的网络是被隔离开的，虚拟机和宿主机是可以互相通信的

    ```markdown
    ## host-only 模式
    config.vm.network :private_network, ip: "11.11.11.11"
    ```

  - Bridge(桥接模式)，该模式下的`VM` 就像是局域网中的一台独立的主机，可以和局域网中的任何一台机器通信，这种情况下需要手动给 `VM` 配 `IP` 地址，子网掩码等

- `hostname`设置

  `config.vm.hostname = "kubernetes"`

- 目录共享

  通过配置来设置额外的同步目录

  ```markdown
  ## 第一个参数是主机的目录，第二个参数是虚拟机挂载的目录
  config.vm.synced_folder  "/Users/haohao/data", "/vagrant_data"
  ```

- 端口转发

  ```markdown
  ## 对宿主机器上 8080 端口的访问请求 forward 到虚拟机的 80 端口的服务上
  config.vm.network :forwarded_port, guest: 80, host: 8080
  ```

### 5、Vagrant 启动虚拟机集群

前面我们都是通过一个 `Vagrantfile` 配置启动单台机器，如果我们要启动一个集群，那么可以把需要的节点在一个 `Vagrantfile` 写好，然后直接就可以通过 vagrant up 同时启动多个 `VM` 组成一个集群。以下示例配置一个 web 节点和一个 db 节点，两个节点在同一个网段，并且使用同一个 box 启动：

```markdown
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.6.0"

boxes = [
    {
        :name => "server01",
        :eth1 => "192.168.1.101",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "server02",
        :eth1 => "192.168.1.102",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "server03",
        :eth1 => "192.168.1.103",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "server04",
        :eth1 => "192.168.1.104",
        :mem => "4096",
        :cpu => "2"
    },
    {
        :name => "server05",
        :eth1 => "192.168.1.105",
        :mem => "4096",
        :cpu => "2"
    }
]

Vagrant.configure(2) do |config|

  config.vm.box = "CentOS-7-x86_64-Vagrant-1905_01"

  boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "vmware_fusion" do |v|
          v.vmx["memsize"] = opts[:mem]
          v.vmx["numvcpus"] = opts[:cpu]
        end

        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
          v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
        end

        config.vm.network :private_network, ip: opts[:eth1]
      end
  end

  config.vm.synced_folder "./labs", "/home/vagrant/labs"
  config.vm.provision "shell", privileged: true, path: "./setup1.sh"

end
```

### 6、Vagrant 常见错误

- Windows 10 RS4 无法完全关闭Hyper-V导致Virtual Box 虚拟机无法启动

  链接：https://www.jianshu.com/p/2e3429d45aea

  ```
  不能为虚拟电脑 WindowsXP 打开一个新任务.
  Raw-mode is unavailable courtesy of Hyper-V. (VERR_SUPDRV_NO_RAW_MODE_HYPER_V_ROOT).
  返回 代码: E_FAIL (0x80004005)
  组件: ConsoleWrap
  界面: IConsole {872da645-4a9b-1727-bee2-5585105b9eed}
  ```

  

  解决办法：

  ```
  以管理员身份运行命令提示符
  执行命令 bcdedit /set hypervisorlaunchtype off
  重启，运行vm即可
  如果想要恢复hyper启动，bcdedit / set hypervisorlaunchtype auto
  ```

  

  ```
  我也遇到了，还有一种可能是Virtualization Based Security导致的，所以即使Hyper-V被禁用了但仍然在运行。
  可以用这个官方工具关闭。
  解压后在Powershell中运行DG_Readiness_Tool_v3.4.ps1 -Disable
  重启后选择关闭该功能就行了
  ```


### 7、启动虚拟机时安装docker，编辑`Vagrantfile`

```bash
Vagrant.configure("2") do |config|
  config.vm.box = "CentOS-7-x86_64-Vagrant-1905_01"
  
  config.vm.provision "shell", inline: <<-SHELL
    sudo yum remove docker docker-client docker-common docker-selinux docker-engine
    sudo yum install -y yum-utils device-mapper-persistent-data lvm2
    sudo yum-config-manager -y --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    sudo yum install -y docker-ce docker-ce-cli containerd.io
    sudo systemctl start docker
  SHELL
end
```
