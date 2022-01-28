# Docker

记录对docker的总结。

- [Docker](#docker)
  - [安装](#安装)
  - [容器](#容器)
    - [容器运行](#容器运行)
    - [容器查看](#容器查看)
    - [容器启停](#容器启停)
    - [容器会话](#容器会话)
    - [容器配置](#容器配置)
    - [容器删除](#容器删除)
  - [镜像](#镜像)
    - [查找和拉取镜像](#查找和拉取镜像)
    - [查看本地镜像](#查看本地镜像)
    - [删除本地镜像](#删除本地镜像)
    - [制作镜像](#制作镜像)
    - [镜像文件](#镜像文件)
  - [互联](#互联)
    - [查看容器内网ip](#查看容器内网ip)
    - [主机-容器端口映射](#主机-容器端口映射)
    - [docker网络](#docker网络)
    - [基于端口映射的远程访问](#基于端口映射的远程访问)
  - [本项目维护相关](#本项目维护相关)

## 安装

- 自动安装

    使用阿里云中提供的脚本进行自动安装，前提是需要curl。这里的curl使用https协议并从指定网址获取内容。[get.docker.com](https://get.docker.com)里的内容就是一段Shell脚本，这段脚本会通过管道传递给下一条命令，也就是bash。然后bash将执行这段脚本。而后面的几个参数也是提供给这个脚本的。
    
    ```c++
    sudo curl -fsSL https://get.docker.com | sudo bash -s docker --mirror Aliyun
    ```

- 手动安装

    如下是使用国内镜像手动安装docker。首先是安装依赖项，然后设置国内源。先从国内镜像添加软件源密钥，接着添加仓库。最后安装docker-ce、docker-ce-cli、containerd.io，安装使用的是国内源。

    ```shell
    sudo apt-get update

    sudo apt-get install \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg-agent \
        software-properties-common

    curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
    sudo apt-key fingerprint 0EBFCD88

    sudo add-apt-repository \
    "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/ \
        $(lsb_release -cs) \
        stable"

    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io
    ```

- 测试是否安装成功

    ```shell
    sudo docker run hello-world
    ```

    如果本地没有hello-word，随后将从[docker hub](https://hub.docker.com)拉取hello-word镜像，接下来为镜像创建一个容器并运行。运行结果是一些关于docker的信息。运行结束后该容器将会进入停止状态，需要手动删除。

## 容器

容器是一种动态实例。类似虚拟机。

### 容器运行

```shell
docker run [-itd] [-p 主机端口:容器端口] [--name 容器名] [镜像名] [命令]
```

|参数|解释|例子|
|:-:|:-|:-:|
|-i|以交互式运行容器|-i|
|-t|为容器分配一个伪终端|-t|
|-d|在后台运行容器；返回容器id|-d|
|-p 主机端口:容器端口|对容器进行端口映射|-p 8080:8000|
|--name 容器名|设置容器名|--name my_container|
|镜像名|表示容器将要运行的镜像|ubuntu:18.04|
|命令|容器开始运行后首先执行的命令|bash|

- 例子

    ```shell
    docker run -itd -p 8080:8000 --name my_container ubuntu:18.04 bash /etc/init.sh
    ```

    - 后台运行一个带伪终端可交互的容器
    - 容器将自身的8000端口与主机的8080端口绑定
    - 容器名为"my_container"
    - 容器将要运行的镜像为ubuntu:18.04；若本地不存在该镜像，则从[docker hub](https://hub.docker.com)拉取
    - 容器开始运行后首先执行的命令是"bash /etc/init.sh"（使用的是容器内的文件路径）
    - 主机执行完成该命令后返回容器id

### 容器查看

```shell
docker ps [-a]
```

|参数|解释|
|:-:|:-|
|空|仅查看运行中的容器|
|-a|查看所有容器，包括未运行的|

### 容器启停

```shell
docker start [容器id|容器名]
```

```shell
docker stop [容器id|容器名]
```

```shell
docker restart [容器id|容器名]
```

- 关于容器名和容器id

    容器名不可简写，容器id可简写为前缀。

- 停止容器带来的影响

    伪终端的所有会话都将退出，即调用Shell中的exit。在run创建的会话中使用exit也能使容器停止。

    停止容器对容器内部不会产生太大影响，后台进程和服务将被暂停；但停止容器会对会话产生影响，例如用户环境变量将被清空 。

### 容器会话

容器会话中一切与会话有关的操作（例如用户环境变量）作用域仅为该会话，会话间不会相互影响。（本质上多会话就是多进程）

- attach

    ```shell
    docket attach [容器id|容器名]
    ```

    这种方式是直接连接容器运行时run创建的会话。这个会话是容器的主会话，退出将导致容器停止。退出该会话的影响等价于停止容器，参考「[容器启停](#容器启停)」。

- exec

    ```shell
    docker exec -it [容器id|容器名] bash
    ```

    以exec创建新的会话。这个会话不是容器的主会话，因此退出该会话不会导致容器退出。但是会话中的用户环境变量等都将被清空。

### 容器配置

配置文件中既包含了容器的名称、状态、镜像等基本信息，也包含了主机配置、驱动、挂载、网络等设置信息。

- 查询某个容器的配置

    ```shell
    docker inspect [容器id|容器名]
    ```

    显示容器配置的json，也就是配置文件中的内容。

- 修改配置

    进入目录`/var/lib/docker/containers/[容器id]/`，其中包含了配置文件。修改配置后，重启容器以生效。

    例如可以修改`hostconfig.json`中的部分字段来改变端口映射关系。

### 容器删除

```shell
docker rm [容器id|容器名]
```

这将删除容器的文件系统、容器配置等与容器相关的部分。但并不会删除容器刚运行时所引用的本地镜像。

## 镜像

镜像是一种静态实例。类似带历史的虚拟机快照。

### 查找和拉取镜像

- 查找

    ```shell
    docker search [搜索内容]
    ```

    在[docker hub](https://hub.docker.com)里按照名称查找所有符合的镜像。

- 拉取

    ```shell
    docker pull [镜像名]
    ```

    从[docker hub](https://hub.docker.com)拉取镜像到本地。

    镜像名的格式既可以是`name`，可以是`author/name`，还可以是`author/name:tag`。

### 查看本地镜像

- 列出镜像

    ```shell
    docker image ls
    ```

    ```shell
    docker images
    ```

    列出所有本地镜像。两命令是等价的。

- 查看镜像详细信息

    ```shell
    docker image inspect [镜像id|镜像名]
    ```

    镜像id的匹配规则和容器id相同。这条命令将返回一个json，记录了镜像的详细信息。

- 查看镜像历史

    ```shell
    docker image history [镜像id|镜像名]
    ```

    查看一个镜像的历史变化。

### 删除本地镜像

- 删除一个

    ```shell
    docker rmi [镜像id|镜像名]
    ```

    ```shell
    docker image rm [镜像id|镜像名]
    ```
    删除某个镜像。两命令是等价的。

- 删除所有未用到的镜像

    ```shell
    docker image prune
    ```

### 制作镜像

- 通过容器

    ```shell
    docker commit -m="[备注信息]" -a="[作者]" [容器id|容器名] [镜像名]
    ```

    将容器提交后将得到一个新的镜像。这个镜像包含了历史信息。

- 通过Dockerfile

    使用Dockerfile可对镜像作出更多设置。

### 镜像文件

镜像文件是镜像在存储介质上的持久化形式。

- 通过容器，生成匿名镜像文件

    - 导出

        ```shell
        docker export [容器id|容器名] > [tar文件路径]
        ```

        直接通过容器生成一个匿名镜像文件。这个文件不包含历史信息，类似单个虚拟机快照，比较小。

    - 导入

        ```shell
        cat [tar文件路径] | docker import - [镜像id|镜像名]
        ```

        导入一个本地匿名镜像文件成为镜像。

        ```shell
        docker import [tar文件路径] [镜像id|镜像名]
        ```

        导入一个异地匿名镜像文件成为镜像。

- 通过镜像，生成完整镜像文件

    - 保存

        ```shell
        docker save [镜像id|镜像名] > [tar文件路径]
        ```

        通过镜像生成一个完整的镜像文件。这个文件包含历史信息，是一或多个虚拟机快照的集合，比较大。

    - 加载

        ```shell
        docker load < [tar文件路径]
        ```

        加载时不需要定义镜像名，因为完整的镜像文件中包含镜像名。


## 互联

### 查看容器内网ip

容器拥有一个在容器所在内网，也就是docker内网中的ip地址。（假定是桥接模式）

- 通过配置文件

    参考「[容器配置](#容器配置)」，查看`/var/lib/docker/containers/[容器id]/config.v2.json`，其中的`IPAddress`字段为容器内网ip地址。

- 通过docker inspect

    docker的inspect命令提供了便捷查看配置的方式，并且我们可以通过其中的`-f`参数来定义输出格式。输入容器名和容器内网ip文本的命令如下：

    ```shell
    docker inspect -f '{{.Name}} - {{.NetworkSettings.IPAddress }}' $(docker ps -aq)
    ```
- 容器内部查看

    容器内部使用`ifconfig`或`ip addr`查看内网ip地址。

    <small>如果没有ifconfig，则安装net-tools；如果没有ping，则安装iputils-ping。</small>

### 主机-容器端口映射

主机和容器之间的统一可由端口映射完成。例如主机外网ip为`22.123.123.123`，内网ip为`192.168.0.4`，容器内网ip为`172.17.0.3`。若将容器8000端口与主机8080端口绑定，则外网`22.123.123.123:8080`等价于内网`192.168.04:8080`等价于容器`172.17.0.3:8000`。

docker的端口映射支持将多个容器端口映射到同一个主机端口，这种情况下网络的表现类似于广播。

- 运行容器时定义

    通过参数`-p [主机端口]:[容器端口]`定义。

- 运行后修改

    首先暂定docker服务：

    ```
    systemctl stop docker
    ```

    参考「[容器配置](#容器配置)」，修改`/var/lib/docker/containers/[容器id]/hostconfig.json`，其中的`PortBindings`字段定义了端口映射关系。

    ```json
    "PortBindings": {
        "8000/tcp": [{
            "HostIp": "",
            "HostPort": "8080"
        }]
    }
    ```

    例如这个字段定义了一个端口映射，将主机的8080映射为容器的8000端口。

    然后修改`/var/lib/docker/containers/[容器id]/config.v2.json`中的`ExposedPorts`。

    最后重启服务：

    ```shell
    systemctl start docker
    ```

### docker网络

docker的network命令提供了关于docker网络的管理。

- 创建网络

    ```
    docker network create [-d 网络驱动] [网络名]
    ```

    创建一个新的docker网络。这个网络是docker使用的虚拟网络，默认是桥接模式。

    参数`-d`定义了网络驱动，目前支持两种，`bridge`是桥接模式驱动，`overlay`是重叠模式驱动。

- 连接与断开

    你可以将docker容器连接到某个网络中。

    - 创建容器时连接

        通过`-net`参数定义容器刚运行时所在的网络，默认值是桥接网络。docker默认就存在几个网络：`bridge`是默认桥接网络、`host`是主机网络、`none`是无网络。这个参数值还可以是通过创建网络创建的自定义网络。

    - 创建容器后连接

        ```
        docker network connect [网络名] [容器id|容器名]
        ```

        将容器连接到某个docker网络。一个容器可以连接到多个docker网络。

    - 断开连接

        ```
        docker network disconnect [网络名] [容器id|容器名]
        ```

        将容器从某个docker网络断开。

- 查看网络

    - 查看一个网络

        ```
        docker network inspect [网络名] 
        ```

        返回为json形式。

    - 列出所有网络

        ```
        docker network ls
        ```

- 删除网络

    ```
    docker network rm [网络名] 
    ```

    删除某个docker网络。

### 基于端口映射的远程访问

- 关于本项目的问题

    本项目中的部署问题涉及多节点，因此肯定需要解决网络互联问题；同时为了方便部署、备份、修改，也需要引入容器化技术。所以最终要解决的就是异地容器互联问题。

- 切入点

    项目的一个核心是配置文件。配置文件中定义了各种服务的地址关系，所有在解决了物理上和虚拟上的网络互联后，仅仅更改这些地址关系即可。

- 思路

    采用docker端口映射方法，把主机与容器在某些端口上等价化处理。考虑内部地址关系可以全部使用`lan_ip:port`表示。这种方法简单有效，但显然不够灵活，因为重启LAN后很可能会导致内网LAN ip发生变动，而这将进一步导致一系列配置文件需要修改，其复杂度等于配置文件中网络地址的出现次数之和。接下来需要考虑如何降低维护的复杂性。

- 关于docker网络的地址动态性问题

    - 问题引入

        同一docker网络中的容器是可以相互连接相互发现的，并且测试发现当容器加入同一自定义docker网络后，在ping和telnet命令中，除了可以使用ip和域名，也可以使用主机名。而容器的主机名在docker网络中就是容器名。
        
        如此一来，似乎可以解决LAN ip变动的问题。但是进一步测试发现主机名在项目中并不能直接使用，所以直接修改项目配置文件中的网络地址为主机名显然不能完全解决这个问题。继续搜索该问题，发现可能可以通过hosts文件解决这个问题，它存储了主机名到ip的映射。接下来的问题就是如何通过hosts去降低维护复杂性。

    - hosts和自定义网络

        来到hosts。当容器处于自定义docker网络后，hosts文件中会自动加入自己在eth3中的ip地址，使用的是容器id的12位缩写。但是hosts中不会自动加入其他处于同一网络下的主机的主机名与eth0中的ip地址。

        然后注意到docker网络中会分配数个容器ip，容器互联用的是eth0中的ip形如`172.17.0.2`，和容器创建顺序有关，是连接docker的默认桥接网络后得到的ip；还有一个eth3中的ip形如`172.19.0.2`，这是容器连接自定义网络后的ip。容器只有连接了自定义docker网络，也就是接入了eth3，才能予以容器名到ip的映射。换言之，加入eth0，即docker默认桥接网络时，不能提供容器名到ip的映射。

        这个例子只是让我们对docker网络有更深刻的理解，能够使我们严格区分自定义网络和默认网络。即便有了自定义网络也仍然无法解决本项目的地址动态性问题，因为docker网络此时仅仅只能提供LAN下单机内部的映射，无法解决远程跨主机的问题。

    - 使用hosts进行手动的地址维护

        测试发现，向hosts中加入主机LAN ip和容器名，就能在外部使用主机端口映射访问服务。也就是说，通过hosts并且不需要自定义docker网络实现容器名到ip的映射。当然这种映射并非直接映射，而是转化为了更高层的主机。

        当然，转化为更高层的好处就是允许我们进行远程访问。原本我们使用的是本地docker维护的内网ip，以172开头；现在我们使用的是LAN维护的内网ip，以192开头，使得访问可以跨主机。

        至此，问题暂时画上句号。我们只需要使用容器名去指代主机LAN ip并将其添加进hosts文件，然后使用`docker_name:port`去替换配置文件中的网络地址就能解决配置文件中的动态地址问题。接下来，复杂性就暂时转移给了hosts文件的维护，其复杂度等同于LAN中的主机数的平方。

        如果继续优化，则是需要将hosts文件的维护过程自动化。

## 本项目维护相关