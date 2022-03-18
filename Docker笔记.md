[toc]

# Docker 的安装

根据官网的提示安装。

注意：Mac 使用了 Linux 虚拟机来安装 Docker，如果想打开虚拟机的目录，可以运行一个 OS，比如：

* 启动后删除：`docker run --rm -it --privileged --pid=host alpine:3.13.4 nsenter -t 1 -m -u -n -i sh`
* 启动后不删除，后台运行：`docker run  -d -it --privileged --pid=host --name os alpine:3.13.4 nsenter -t 1 -m -u -n -i sh`

如果是 Linux：

* systemctl start docker：启动 Docker 服务（stop 关闭，status 查看状态，restart 重启）
* docker version：看看是否成功，或者 docker info
* systemctl enable docker：每次启动服务器，自动启动 Docker

***

加速地址：

Docker中国区官方镜像
https://registry.docker-cn.com

网易
http://hub-mirror.c.163.com

ustc
https://docker.mirrors.ustc.edu.cn

中国科技大学
https://docker.mirrors.ustc.edu.cn

阿里云容器 服务
https://cr.console.aliyun.com/
首页点击“创建我的容器镜像” 得到一个专属的镜像加速地址，类似于“https://1234abcd.mirror.aliyuncs.com”

添加阿里云镜像加速的方式 - Linux：

```
1. 安装／升级Docker客户端
推荐安装1.10.0以上版本的Docker客户端，参考文档docker-ce

2. 配置镜像加速器
针对Docker客户端版本大于 1.10.0 的用户

您可以通过修改daemon配置文件/etc/docker/daemon.json来使用加速器

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://1234abcd.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

添加阿里云镜像加速的方式 - Mac：

```
1. 安装／升级Docker客户端
对于10.10.3以下的用户 推荐使用Docker Toolbox

Mac安装文件：http://mirrors.aliyun.com/docker-toolbox/mac/docker-toolbox/

对于10.10.3以上的用户 推荐使用Docker for Mac

Mac安装文件：http://mirrors.aliyun.com/docker-toolbox/mac/docker-for-mac/

2. 配置镜像加速器
针对安装了Docker Toolbox的用户，您可以参考以下配置步骤：

创建一台安装有Docker环境的Linux虚拟机，指定机器名称为default，同时配置Docker加速器地址。

docker-machine create --engine-registry-mirror=https://1234abcd.mirror.aliyuncs.com -d virtualbox default
查看机器的环境配置，并配置到本地，并通过Docker客户端访问Docker服务。

docker-machine env default
eval "$(docker-machine env default)"
docker info
针对安装了Docker for Mac的用户，您可以参考以下配置步骤：

在任务栏点击 Docker Desktop 应用图标 -> Perferences，在左侧导航菜单选择 Docker Engine，在右侧输入栏编辑 json 文件。将

https://1234abcd.mirror.aliyuncs.com加到"registry-mirrors"的数组里，点击 Apply & Restart按钮，等待Docker重启并应用配置的镜像加速器。

也就是：{"registry-mirrors":["https://1234abcd.mirror.aliyuncs.com"]}
```

***

其他添加加速地址的方式：

* docker --registry-mirror=https://registry.docker-cn.com
* systemctl restart docker（如果是 json 方式还需要 sudo systemctl daemon-reload）
* docker info（检查）

还可以参考：https://www.runoob.com/docker/docker-mirror-acceleration.html 和 https://www.cnblogs.com/nhdlb/p/12567154.html

# Docker 基础

Docker 基础概念：

* 【宿主机】<u>安装并运行 Docker 引擎</u>
* 【Docker 引擎】<u>创建沙箱（沙箱也叫容器）</u>
* 【每个沙箱】<u>运行着进程</u>
    * 沙箱 1 运行进程 1，沙箱 2 运行进程 2

## Docker 原理 - 利用了 Linux 的特性

利用 Linux 的特性：

* Namespaces 隔离：
    * 进程树
    * 网络接口
    * 挂载点
    * 进程间通信
* CGroups 隔离共享资源，如：
    * CPU
    * 内存
    * 磁盘 I/O
    * 网络带宽配额
* Unionfs 构成：
    * 文件系统
    * 镜像

## Docker 原理 - Namespaces

Namespaces 实现功能：

* 隔离进程
    * 在容器 1 内，不应该看到容器 2 内的进程
    * 所以需要将沙箱 1 和沙箱 2 做成两个不同的 Namespaces
    * 也就是说，在*宿主机*上运行 ps -aux 可以看到所有进程，但是进入 *容器1* 内，再运行 ps -aux 就只能看到容器 1 的进程，无法看到*容器2*的进程
* 隔离网络接口（默认：网桥模式）
    1. 给每个容器做了一块虚拟网卡
    2. 将每个容器的虚拟网卡，全部接到 *Docker0* 上
    3. *Docker0* 通过 iptables 和*宿主机*的网卡相连来完成通信

## Docker 原理 - CGroups

CGroups 实现功能：

* 隔离共享资源
    * 一般来说，进程 1 只能使用 CGroup1 内的资源
    * 也就是，限制了 1 个容器能够使用资源的量

## Docker 原理 - Unionfs

Unionfs 实现功能：

* Unionfs 就是「联合文件系统」：
    * 基础
        * 假设有 Ubuntu1 系统和 Ubuntu2 系统
        * 把两个系统之间相同的部分，设置为<u>只读层</u>
        * 把不同的部分，设置为<u>可写层</u>
    * 例子
        * 假设有 3 个层
            * fs3 里面有可写的文件 /a/b/c.txt
            * fs2 里面有只读的目录 /a/b
            * fs1 里面有只读的目录 /a
        * 使用起来的感受是 1 层
            * 看起来就是 /a/b/c.txt
            * 可以把 c.txt 删掉或者修改
            * 无法删除 /b 目录，会提示无权限
    * 总结
        * 看起来有 Ubuntu1 系统和 Ubuntu2 系统
        * 但是，相同的地方共用了*只读文件系统层1*
        * 不同的地方，分别有*可写文件系统层1*和*可写文件系统层2* 
* 在 Docker 中， *只读层（相同的部分）* 被称为<u>镜像</u>， *可写层（不同的部分）* 就是<u>容器</u>

Unionfs 延伸出 Docker 的使用方法：

1. <u>Pull（拉取）一个 Image（镜像）</u>：获取相同的部分/只读层
2. <u>创建一个 Container（容器）</u>，即 **创建一个可以写的目录，把这个 Writable 的目录叠加地放在 Image 上** ：创建不同的部分/可写层
3. <u>在容器中运行命令</u>：**虚拟机启动后会一直运行，Docker 是用来运行命令，一旦命令结束，容器就会退出**

## Docker 的架构

总体框架：

* Client
    * 用来获取命令
    * 根据命令和 Server 的 Daemon 来通信
* Server
    * 拉取镜像和创建容器：
        1. Server 拉取 Image（如果本地有该 Image，就使用本地的）
        2. 创建可写的文件系统层
        3. 用 Unionfs 把上述东西挂载起来
    * Daemon 后台进程：
        * 「Server 内的 Daemon」用来操作「Server 内的 Containers（容器）和 Images（镜像）」
        * Daemon 会把 Containers 和 Images 执行的结果返回给 Client
* Registry（也称为 Repository）：仓库可看成一个代码控制中心，用来保存镜像


注意：【Server】有一个 *Daemon 后台进程* ，这个 *Daemon* 用来和【Client】进行通信，我们必须把【Server】运行起来，【Client】才能通信运作

# Docker 命令

注意，命令在 Client 端，如果 Server 端没有开启，就无法执行大部分命令

可以使用 ID，也可以自定义 --name 后使用名称

## 容器管理

`docker run [镜像名]` 根据该镜像，创建新的容器，也就是运行该镜像上的容器（如果没有该镜像，就会自动执行 pull 来拉取镜像）

* 具体的看[官网](https://hub.docker.com/)的介绍，有些需要 `-e` 定义环境

`docker run --name [自定义镜像名] [原镜像名或ID]`

`docker run -p 8080:80 nginx` 将该 nginx 的 80 端口，映射为 docker 的 8080 端口（也就是本机的 8080 可以访问 nginx 的 80 端口）

`docker run -v /usr/share/index.html:ro -d nginx` 后台启动 nginx，读取宿主机的 /usr/share/index.html 静态文件，并且设定为 ro（read only）

`docker start [容器ID]` 启动已经存在的一个容器

`docker restart [容器ID]` 重启容器

`docker stop [容器ID]` 停止该容器

`docker kill [容器ID]` 强制停止该容器

`docker run --rm [容器名]` 因为每运行一次 `run` ，都会创建一个容器，所以使用这个命令，可以 `run`  后自动删除该容器

`docker rm -f [容器ID]` 强制删除该容器

`docker rm -f $(docker ps -qa)` 通过 -qa 获取所有容器的 ID，然后执行删除操作。也就是删除所有容器。

`docker container prune` 删除停止的、没有用的容器（因为每运行一次，都会创建一个容器）

`docker exec -it [容器ID] /bin/sh` 以交互的方式，用 sh 命令进入该容器（可以不写 /bin/sh，而是 sh；很多时候还可以使用 bash 或 /bin/bash），退出的时候可以使用 exit 命令

`docker commit [容器ID] -m "{信息}" -a "{作者}" [新镜像名]:[新镜像tag] ` 保存当前容器及其修改的内容，生成新的镜像，客户端会返回新镜像的 ID。

* 我们可以使用 `docker tag [镜像id] [镜像名称]:[镜像标签]` 来给这个新生成的镜像命名和添加标签

`docker build` 使用 Dockerfile 创建镜像，解决了 docker commit 只是生成了结果，没有保存步骤的问题

## 镜像管理

`docker search [镜像名]:[标签/版本号]` 搜索镜像，或者去 https://hub.docker.com/ 搜索

`docker pull` 和 `docker push` 用来传输镜像

`docker tag` 给不同版本或不同类型的镜像添加标签，下次拉取的时候，就可以通过标签名来操作

`docker images` 查看所有镜像

`docker images [过滤词]` 查看含有该词的镜像

`docker image rm -f [镜像ID]` 或 `docker rmi [镜像ID]` remove 该 image（除了 镜像 ID，也可以使用 镜像名:tag 来删除）

`docker rmi -f $(docker images -q)` 通过 -q 获取所有镜像 ID，然后执行删除操作。也就是删除所有镜像。

`docker image prune` 删除不用的镜像

`docker image prune -a` 删除所有没有被其他镜像引用的镜像

`docker load -i [本地镜像文件的路径.tar]` 提取被压缩的镜像，载入到 Docker（Mac 上，将 tar 文件放到 /Users/$(用户名) 目录下即可）

`docker save [镜像标识] -o [输出的路径及文件名.tar]` 压缩镜像（如果没有加路径，Mac 默认放到 /Users/$(用户名) 目录下）

## 信息和状态查询

`docker diff [容器ID]` 查看容器的修改

`docker port [容器ID]` 查看正在运行的容器的端口映射

`docker logs [容器ID]`

* `docker logs -f [容器ID]` 实时显示日志
* `docker logs -ft [容器ID]` 实时显示日志，并加入时间戳
* `docker logs --tail 5 [容器ID]` 查看日志最后 5 行

`docker stats`

`docker version`

`docker top [容器ID]` 查看容器内部运行的进程

`docker inspect [容器ID]` 查看容器的元信息

`docker inspect --format '{{ .NetworkSettings.IPAddress }}' [容器ID]` 查看该容器的 IP 地址

# Docker 和宿主机的数据交换和文件共享

复制资源：

* `docker cp [原文件路径] [将文件复制到哪个路径]`
    * `docker cp [容器id]:[容器内部的路径] [本地路径]`：从容器复制到宿主机
    * `docker cp [本地路径] [容器id]:[容器内部的路径]`：从宿主机复制到容器

Volume（数据卷）：

* 实现容器和宿主机之间的文件共享

如果想在宿主机上存留持久化数据，就需要<u>数据卷</u>，其实现原理：

* 在宿主机的 Filesystem 里面分配一块 Docker Area
* Container（容器）通过 bind mount 挂载连接 Docker 目录以外的目录，也就是外部目录（宿主机上的 Filesystem）
* 此时，Container 可以操作系统外部目录

还可以操作 Memory（内存）：

* 内存也是一种文件
* 可以通过 tmpfs mount 来挂载内存

数据卷管理：

bind mount 方式（不推荐）

* `docker run -v [宿主机绝对目录]:[容器内目录] [容器id或名称]`
* 如果 host 机器（宿主机）上的目录不存在，docker 会自动创建该目录
* 如果 container 中的目录不存在，docker 会自动创建该目录
* 如果 container 中的目录已经有内容，那么 docker 会使用 host 上的目录将其覆盖掉

volume 方式 - 启动的时候，自定义（推荐）

* `docker run -v [自定义名称]:[容器内目录] [容器id或名称]`
* 这种方式，宿主机的目录就为 `/var/lib/docker/volumes/{自定义名称}` 
    * 如果没有 `[自定义名称]:` 就会自动创建一个随机名称（可以使用 docker volume inspect 查看具体的路径等信息）
    * 可以通过 `find / -name [自定义名称]` 来查找位置，也可以通过 `docker inspect [数据卷标识]` 查看位置
* 如果 volume 是空的，而 container 中的目录内有内容，那么 docker 会将 container 目录中的内容拷贝到 volume 中
* 但是，如果 volume 中已经有内容，则会将 container 中的目录覆盖

volume 方式 - 自定义后，启动的时候再挂载（推荐）

* `docker volume create [自定义名称]`
* `docker run -v [已经定义好的名称]:[容器内目录] [容器id或名称]`

其他 Volume 相关指令（命令）：

* `docker volume ls` 查看所有
* `docker inspect [数据卷名称]` 或 `docker volume inspect [数据卷名称]` 查看改数据卷信息
* `docker volume rm ...` 删除
* `docker volume prune` 清理所有数据卷

Dockerfile 中的 VOLUME：

* `VOLUME /foo`
    * Docker 运行时创建一个匿名的 Volume
    * 将此 Volume 绑定到 container 的 /foo 目录中
    * 如果 container 的 /foo 目录下已经有内容，则会将该目录下的内容拷贝的 volume 中
* Dockerfile 中的 Volume 每次运行一个新的 container 时，都会为其自动创建一个匿名的 volume
    * 如果需要在不同 container 之间共享数据，那么依然需要通过 `docker run -v [自定义名称]:[容器内目录]` 的方式将【容器内目录】 中数据，存放于指定的【自定义名称】的 volume 中

参考资料：[Docker学习笔记（6）——Docker Volume](https://www.jianshu.com/p/ef0f24fd0674)

# Docker 网络

## Docker 网络基础

> 默认的网络叫做 docker0，平时启动的容器都在 docker0 的网络环境下运行

Docker 网络模式：

* 平时使用 `docker run` 的时候，会默认添加 `--net bridge` 或 `--net=bridge` 参数：
    * 这种就是默认使用 docker0 的网络环境，不过域名不能访问
    * ~~可以通过 `--link` 来打通连接（不推荐）~~
* 可以使用参数 `--net=host` 或 `--net host` 来修改模式为 host 模式（host 模式可以和宿主机共用网络环境，不过不推荐使用）
* 还可以 `--net=container:[某个容器标识]` 来让 新启动的容器 共用 某个容器 的网络环境（不过，新启动的容器只能使用该网络环境指定的端口）
* 参考：[Docker 的 四种网络模式](https://blog.csdn.net/huanongying123/article/details/73556634)或[用印象笔记打开](https://app.yinxiang.com/shard/s72/nl/16998849/f6c9ead3-3164-4337-a514-564632f68d06/)

基础命令：

* `docker network ls` 查看 Docker 的所有网络信息
* `docker network inspect [网络标识]` 查看具体网络信息

## 自定义网络

推荐使用自定义网络：

`docker network create --subnet 192.168.0.0/24 --gateway 192.168.0.1 mynet`

* 默认会添加 `--driver bridge` 所以可以不写，用于指定模式
* `--subnet 192.168.0.0/24` （不写的话，会自动配置一个）
	* 用于设置子网，其中的 `/24` 是 CIDR 值（一个 CIDR 值对应一个子网掩码，然后对网络就行分段）
		* `/24` 对应的是 255.255.255.0
		* ~~`192.168.0.0/24` 就代表了 IP 为 192.168.0.0 至 192.168.0.255~~
	* 如果是 `192.168.0.0/16` ，最多到 255.255.0.0
		* ~~该网络上可以有 2^16-2，也就是 65534 个主机~~
		* ~~IP 范围：192.168.0.1~192.168.255.254~~
* `--gateway 192.168.0.1` （不写的话，会自动配置一个）
	* 这个是网关，要比 subnet（子网）大 1 个点（子网 192.168.0.0，网关 192.168.0.1）
	* 所有都经过它
* `mynet` 自定义网络的名称

使用 `docker run` 的时候，添加参数 `--net [自己定义完成的网络的名称]` 即可让该容器在该网络状态下运行：

`docker run --net mynet --name myredis redis:5.0.5`

* 表示启动 myredis 容器，使用自定义的 mynet 网络环境
* 注意，此时会自动将 myredis 容器的 IP 的 host 设置为 myredis，也就是说，可以通过 myredis:6379 来访问该 redis 容器
* 相当于修改了 host 文件，让 myredis 来映射原来的 IP

## 联通网络

如果有 docker0 和自定义的 mynet 这两个网络，*Container0* 运行在 *docker0* 上，*Container1* 运行在 *mynet* 网络上。

如果要将名为 *Container1* 的容器运行在 *docker0* 的网络环境下：

`docker network connect docker0 Container1`

此时，*Container1* 可以同时使用 *mynet* 和 *docker0* 的网络环境，所以 *Container1* 可以联通 *Container0* 。

但是，此时 *Container0* 还是只存在于 *docker0* 中，所以无法去请求 *Container1*，只有运行下面这个命令，才能互相打通：

`docker network connect mynet Container0`

# Dockerfile
## Dockerfile 基础

Dockerfile 所在的目录（在 host 上的目录）被称为 context（上下文目录）：

* A build’s *context* is the set of files located in the specified *PATH* or *URL*
* The build process can refer to any of the files in the context 
* For example, your build can use a *COPY* instruction to reference a file in the context

Dockerfile 构建镜像的原理：

* 打包 context 下所有的文件（假设是根目录，就会打包所有根目录的内容），发送到 Docker Server
* 然后根据 Dockerfile 的指令，每一行都生成一个 image
* 最后产生出最终的 image（构建时，每一行生成的 image 会放在缓冲区）
* cache 机制
    * Dockerfile 的每一行指令，都会在缓存区（cache）生成一个临时的镜像（image）
    * 如果某一行指令出错了，下次再修改、或者（没有出错，而是要）重新创建的时候，就可以复用缓存区已经生成的镜像
    * 如果构建（build）的时候，不想使用上次缓存的镜像，可以加上 `--no-cache`（Do not use cache when building the image）

构建时和运行时的区别：

* 构建时：构建一个镜像，不变的内容要先固定好
* 运行时：运行的容器需要的内容，可以在构建后，通过网络，拉取代码仓库内最新的代码（比如，配置文件），接着执行

## 编写 Dockerfile

`docker build -t [名称和标签] [context目录]`

* 运行 Dockerfile 来 build 镜像
* `docker build -t myapp .` ：根据当前目录，创建名称为 myapp:latest 的镜像（Dockerfile 文件在当前目录下，比如 *./myapp.Dockerfile* 或者 *./Dockerfile* ）

如果 context 放置了和构建当前镜像无关的文件，需要编写 `.dockerignore` 来忽略这些文件：

* `docker build` searches for（自动搜索，不需要指定） a `.dockerignore` file relative to the Dockerfile name
* For example, running `docker build -f myapp.Dockerfile` . will first look for an ignore file named `myapp.Dockerfile.dockerignore`

`FROM` ：基础的镜像

`RUN` <u>构建镜像时需要运行的指令</u>：

* exec 命令形式：参数1 是 executable，剩下的参数是 params，也就是普通的命令行模式：`RUN apk add --update vim`
* 还有一种是 `RUN ["apt-get", "install", "python3"]`
* 也可以指定使用哪个 shell 来执行

`EXPOSE` 该镜像在运行为容器时，对外暴露的端口号：

* `EXPOSE 80` 默认是 `EXPOSE 80/tcp`
* `EXPOSE 80/udp` 可以暴露 udp 端口

`WORKDIR` 创建容器后，终端默认的路径：

* `WORKDIR /` ：使用根目录作为路径
* 还可以指定多个 WORKDIR。假设要以 `/data/test` 为路径：
	* `WORKDIR /data` （绝对路径）
	* `WORKDIR test` （相对于上一个路径的相对路径，也可以写其他的绝对路径）
	* 这里相当于先设定为 /data，然后切换到 test，这两个指令之间可以添加其他指令，来灵活配置，比如先设定目录为根目录，再 ADD 一个 test.tar.gz 文件（解压缩后的文件为 test）到根目录，最后设定目录为 test 目录

`COPY` 和 `ADD` 用于添加文件：

* `COPY` ：只支持从 context 目录下拷贝文件到镜像内
	* `COPY [宿主机上的context目录的原文件1] [原文件2] ... [镜像内的目标路径]`
	* `COPY a.txt b.txt /data` ：拷贝 context 路径下的 a.txt 和 b.txt 文件，到镜像内的 /data 路径（假设 context 在 host 的根目录，就是把 host 下的 /a.txt 和 /b.txt 复制到镜像内的 /data 中）

* `ADD`：支持 `COPY` 的操作，还支持 URL 请求下载文件、解压缩文件等
	* `ADD [请求的网址] [镜像内的目标路径]` 自动发送 GET 请求到网址的资源（一般是下载的资源）到镜像内
	* `ADD [context目录下的压缩文件.tar.gz] [镜像内的目标路径]` 解压缩并放入镜像内 

`ENV` 定义环境变量：

* `ENV [大写的key]=[小写的value]` 或者 `ENV [大写的key] [小写的value]`
* `ENV JAVA_HOME=/usr/local/openjdk-17` （摘抄自 openjdk）
* 还可以自定义一个目录，然后使用 `$` 符号来引用这个目录：
	* `ENV BASE_DIR /data` 
	* 然后 `ADD xxx.tar.gz $BASE_DIR` 以及 `VOLUME $BASE_DIR/xxxdata`

`VOLUME` 镜像层面的挂载：

* This will initialize the newly created volume with any data that exists at the specified location within the base image
*  `VOLUME /data` ：这样可以在镜像内创建 /data 目录用于挂载：
	* 运行容器时还可以继续挂载 mydata 这个 volume 到该容器的 /data 位置：
	* `docker run -d -v mydata:/data xxxx`
* 格式为：`VOLUME [镜像内路径1] [路径2]...`

* 参考资料：[VOLUME 定义匿名卷](https://yeasy.gitbook.io/docker_practice/image/dockerfile/arg) 和 [官网介绍](https://docs.docker.com/engine/reference/builder/#volume)

`ENTRYPOINT` 和 `CMD` 指定根据镜像生成的<u>容器启动时，运行的命令</u>：

* 结论：**`ENTRYPOINT` 填写不能修改的命令， `CMD` 填写可以修改的命令（修改 CMD 命令的时候，直接 `docker run [镜像标识] [覆盖CMD的新命令]` 来启动容器）**
	* 假设 JDK 的镜像名称为 this.jdk，它里面有一个 test.jar 和 pro.jar，默认要启动 test.jar，但是有时候需要运行 pro.jar
	* 可以 `ENTRYPOINT ["java", "-jar"]` 和 `CMD ["test.jar"]`
	* 运行为容器的时候，如果需要的是 test.jar，直接 `docker run this.jdk` 就行了
	* 如果不要运行 test.jar，而是 pro.jar，就使用  `docker run this.jdk pro.jar`
* 区别 - 例子：
	* 如果写了 `CMD ["ls", "-a"]`，通过 `docker run` 命令运行的时候，只会执行 `ls -a` 
		* 如果想达成 `ls -la` ，就没办法直接启动的时候使用 `docker run xxx -l` ，也就是添加 `-l` 的参数来完成目的了
		* 因为启动的命令行里面的 `-l` 会替换 Dockerfile 定义的  `CMD ["ls", "-a"]` ，也就是没有 `ls -a` ，而是变成了直接 `-l`
		* 此时要正常执行，只能在命令行完整地写 `docker run xxx ls -la` 
	* 换句话说，`CMD` 相当于默认的指令，如果我们<u>在命令行的末尾输入自己的指令</u>来运行，就可以达成<u>覆盖 `CMD` 指令</u>的效果
	* 如果写的是 `ENTRYPOINT ["ls", "-a"]` 的话，启动的时候，可以<u>在命令行的末尾添加追加的参数，而不会被覆盖</u>：
		* 比如想完成  `ls -la` 可以在命令行启动容器的时候，直接 `docker run xxx -l` 来追加参数
		* 也就是说，可以在启动容器的命令行内，追加参数，上面的例子在追加参数后变为了 `ENTRYPOINT ["ls", "-a", "-l"]`
	* 如果 `ENTRYPOINT` 需要覆盖，就在命令行启动 `docker run --entrypoint=[新的execute命令] [容器标识] [新的参数]`
* [区别 - 官网定义](https://docs.docker.com/engine/reference/builder/#understand-how-cmd-and-entrypoint-interact)：
	* Dockerfile should specify at least one of `CMD` or `ENTRYPOINT` commands.
	* `ENTRYPOINT` should be defined when using the container as an executable.
	* `CMD` should be used as a way of defining default arguments for an `ENTRYPOINT` command or for executing an ad-hoc command in a container.
	* `CMD` will be overridden when running the container with alternative arguments.
	* An `ENTRYPOINT` allows you to configure a container that will run as an executable.
	* The main purpose of a `CMD` is to provide defaults for an executing container
* 注意：
	* If `CMD` is used to provide default arguments for the `ENTRYPOINT` instruction, both the `CMD` and `ENTRYPOINT` instructions should be specified with the JSON array format.
	* 也就是说，用  `["命令"]` 的方式绝对没有错
* 参考资料：[CMD](https://docs.docker.com/engine/reference/builder/#cmd) 和 [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint)

***

IDEA 中的 Dockerfile：

IDEA 连接 Docker，参考 [Docker | IntelliJ IDEA - JetBrains](https://www.jetbrains.com/help/idea/docker.html)

在 context 的目录（在 target 下自己创建一个目录，然后把 jar 包移动过去）右键 New -> File，输入 Dockerfile 生成一个文件，IDEA 会自动识别并提示：Dockerfile detection

~~废弃的笔记：~~

~~在 IDEA 状态栏 Tools -> Deployment -> Browse Remote Host，打开远程连接部署的选项~~

~~点击 `...` 弹出 Add Server 选项卡，Name 填写 docker0 的 IP 地址，Type 选择 SFTP，然后点击 OK 按钮~~

~~查询 docker0 的 IP 地址：~~

~~1. `docker inspect bridge` ，然后找到 IPAM 里面的 Gateway~~

~~2. 在 Linux 的命令行输入 `ifconfig` ，找到 docker0 的 inet addr~~

~~然后在 SSH configurations 那里点击 `...` 进入详细设置的选项卡，点击 `+` 按钮添加，Host 填写 IP，~~

# Docker-compose

## Docker-compose 基础

为什么要使用多容器方案？

* 单一容器内如果装有多个应用，更新的时候会很麻烦。比如说，一个容器内有 MySQL 和 Redis，更新的时候整个镜像都需要更新
* 单一容器还容易造成依赖冲突。比如说，这个 Redis 只支持 Java 11，而该容器内的 MySQL 只支持 Java 8，就无法使用了
* 多容器可以让每个容器都依赖于自己的独立环境，而且升级的时候不会产生冲突

产生的问题：

* 互相怎么访问
* 顺序依赖问题：需要先启动 Redis，在启动 Jar 包，不然就会报错

Compose 核心概念：

* service（服务）：一个服务代表一个容器，一个项目可以有多个服务
* project（项目）：一组关联的服务组成的一个完整的业务单元（抽象的概念）

## 使用 Docker-compose 

Mac 自带 Docker-compose，而 Linux（只支持 64-bit Linux）需要自己安装（参考[官方方法](https://docs.docker.com/compose/install/)）：

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version
```

***

编写 `docker-compose.yml`：

> 假设 yml 文件的目录为：
>
> /hello/docker-compose.yml
>
> 假设 Dockerfile 和 jar 包的目录为：
>
> /hello/myjar/Dockerfile（定义了 EXPOSE 8081）
>
> /hello/myjar/test.jaar

```yaml
version: "3" # 使用哪个版本的语法
services:
  web: # 自定义的唯一服务名，可以随便改。注意： 服务名也是这个服务的 IP 的 host
    container_name: web # 相当于 --name web（尽量与服务名一致）
    build: # build 一个 Dockerfile 为镜像，然后启动该镜像
      context: myjar # 指定 context 目录（也可以是绝对路径 /hello/myjar）
      dockerfile: Dockerfile # 确定 Dockerfile 的名称，也可以不写这个
    ports:
      - "8080:8081" # 镜像及其容器暴露了 8081 端口，将其映射到宿主机的 8080 端口
    networks:  # 定义当前服务使用哪个网络配置（要使用这个配置文件的 networks 来创建）
      - mynet
    depends_on:
      - mysql
  mysql:
    image: mysql:5.7.33 # 需要 pull 的镜像
    container_name: mysql
      ports:
      - "3306:3306" # "[宿主机端口]:[容器内端口]" 这个列表可以写多个
      - "9090-9091:8080-8081" # 还可以这样指定多个
    volumes:
      # 挂载到 mysqldata（要使用这个配置文件的 volumes 来创建）
      - "mysqldata:/var/lib/mysql" # 如果是绝对路径，就是宿主机上的目录
    environment: # 可以直接指定环境
      MYSQL_ROOT_PASSWORD: rootroot # 设置密码为 rootroot
      env_file: # 也可以通过 .env 文件导入环境
        - ./common.env # .env 文件必须为：[大写的key]=[小写的value]
        - ./apps/web.env # 如果和前面定义的环境 key 有冲突的地方，
      - /opt/runtime_opts.env # 就以最下面这个 value 为准
        depends_on: # 定义启动当前服务需要的依赖服务
          - redis # 注意，一定要写服务名，而不是容器名
        networks:
          - mynet
  redis:
    image: redis:5.0.5
    # 省略一些配置...
    command: ["redis-server", "--appendonly yes"] # 相当于 CMD
    entrypoint: # 就相当于 ENTRYPOINT
volumes: # 创建数据卷
  # 假设 yml 配置文件所在的目录为 /hello，可以理解为项目名是 hello
  # 如果没有 external 的内容，该 volume 的名称就为 hello_mysqldata
  mysqldata: # 除了这样直接声明，也可以写成：mysqldata:{}
    external: true
    # external 为 true 表示使用自定义的外部卷名，也就是直接为 mysqldata
    # 但是如果定义了 external，就必须提前创建好改 volume
networks: # 创建网络配置
  # 如果项目名为 hello，也就是 yml 配置文件在 /hello 目录下，这个网络配置的名称就是 hello_mynet
  # 也可以像 volumes 那样指定 external 为 true 来更改名称为 mynet，但是这样要提前创建
  mynet: # 如果下面什么都不写的话，有使用默认的 bridge 配置
    ipam:
      driver: default # 也就是 bridge
      config:
        - subnet: "172.16.238.0/24"
        - subnet: "2001:3984:3989::/64"
```

## 启动 Docker-compose 的一些指令

在含有 `docker-compose.yml` 的目录下，后台运行命令（`-d` 参数必须在 `up` 后面）：

```bash
docker-compose up -d
```

查看日志（用法参考 docker 开头的命令）：

```bash
docker-compose logs [服务名...可以有多个，如果不写表示全部]
```

清除之前的 docker-compose 和缓存，并重新启动：

```bash
docker-compose down && docker-compose build --no-cache && docker-compose up
```

如果需要指定 yml 配置文件来后台运行，需要（注意，`-f` 命令一定要在其他命令行参数的前面）：

```bash
docker-compose -f [配置文件名称].yml up -d
```

如果只运行其中服务名为 redis01 和 mysql01 的服务（指定启动的服务如果有依赖，那些依赖的服务也会被启动）：

```bash
docker-compose up redis01 mysql01
```

假设配置文件的路径为 `/hello/docker-compose.yml` ，那么项目名就是 hello；如果不想项目名命名为 hello，而要把项目名命名为 project1：

```bash
docker-compose -p project1 up
```

停止该项目下的所有服务，并移除网络（但是不移除数据卷）：

```bash
docker-compose down
```

进入服务名为 redis01 的容器：

```bash
docker-compose exec redis01 bash
```

查看该项目中所有的服务和查看所有项目进程：

```bash
docker-compose ps [可以指定服务名]
docker-compose top [可以指定服务名]
```

启动、重启和停止项目中的服务：

```bash
docker-compose start [服务名...可以有多个，如果不写就是全部]
docker-compose restart [服务名...可以有多个，如果不写就是全部]
docker-compose stop [服务名...可以有多个，如果不写就是全部]
```

# 清理 Docker

查看 Docker 的磁盘使用情况：

```bash
docker system df
```

清理磁盘相关数据：

```bash
docker system prune
docker system prune -a
```

清理数据卷：

```bash
docker volume prune
```

