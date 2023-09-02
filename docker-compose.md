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
    container_name: web # 相当于 --name web（不是服务名！是容器名称！）
    build: # build 一个 Dockerfile 为镜像，然后启动该镜像
      context: myjar # 指定 context 目录（也可以是绝对路径 /hello/myjar）
      dockerfile: Dockerfile # 确定 Dockerfile 的名称，也可以不写这个
    ports:
      - "8080:8081" # 镜像及其容器暴露了 8081 端口，将其映射到宿主机的 8080 端口
    networks: # 定义当前服务使用哪个网络配置（要使用这个配置文件的 networks 来创建）
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
    command: [ "redis-server", "--appendonly yes" ] # 相当于 CMD
    entrypoint: # 就相当于 ENTRYPOINT
volumes: # 创建数据卷：如果提前创建了，就会使用提前创建的；如果没有创建，会自动创建新的
  # 假设 yml 配置文件所在的目录为 /hello，可以理解为项目名是 hello
  # 如果没有 external 的内容，该 volume 的名称就为 hello_mysqldata
  mysqldata: # 除了这样直接声明，也可以写成：mysqldata:{} 或 mysqldata:
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

在含有 `docker-compose.yml` 的目录下，后台运行命令（`-d` 参数必须在 `up` 后面）：

```bash
docker-compose up -d
```

实时查看当前目录下的 docker-compose 的服务日志（用法参考 docker 开头的命令）：

```bash
docker-compose logs -f [服务名...可以有多个，如果不写表示全部]
```

如果需要指定 yml 配置文件来后台运行，需要（注意，`-f` 命令一定要在其他命令行参数的前面）：

```bash
docker-compose -f [配置文件名称].yml up -d
```

如果只运行其中服务名（注意，不是容器名，是服务名）为 redis01 和 mysql01 的服务（指定启动的服务如果有依赖，那些依赖的服务也会被启动）：

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

删除容器中已经停止的服务（如果不添加服务名，表示删除当前目录下的所有 docker-compose 服务），`-f` 表示强制删除，`-v` 表示同时删除匿名的数据卷（自定义的数据卷不会被删除）：

```bash
docker-compose rm -f [服务名...]
docker-compose rm -v [服务名...]
docker-compose rm -fv [服务名...]
```

