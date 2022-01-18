# Docker

记录对docker的总结。

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

容器会话中一切与会话有关的操作（例如用户环境变量）作用域仅为该会话，会话间不会相互影响。

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
    sudo docker image inspect [镜像id|镜像名]
    ```

    镜像id的匹配规则和容器id相同。这条命令将返回一个json，记录了镜像的详细信息。

- 查看镜像历史

    ```shell
    sudo docker image history [镜像id|镜像名]
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

    暂不涉及。

### 镜像文件

镜像文件是镜像在存储介质上的持久化形式。

- 通过容器，生成匿名镜像文件

    - 导出

        ```shell
        sudo docker export [容器id|容器名] > [tar文件路径]
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
        sudo docker save [镜像id|镜像名] > [tar文件路径]
        ```

        通过镜像生成一个完整的镜像文件。这个文件包含历史信息，是一或多个虚拟机快照的集合，比较大。

    - 加载

        ```shell
        docker load < [tar文件路径]
        ```

        加载时不需要定义镜像名，因为完整的镜像文件中包含镜像名。


## 互联

## 维护流程