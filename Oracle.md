# Oracle 学习笔记

## 安装部署

摘抄自：[Docker安装Oracle_11g](https://www.jb51.net/article/208976.htm) （[原文地址](https://blog.csdn.net/qq_42021376/article/details/115444547?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167766259016800227481245%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=167766259016800227481245&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-115444547-null-null.blog_rank_default&utm_term=oracle&spm=1018.2226.3001.4450)）

**1.拉取oracle_11g镜像**

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/helowin/oracle_11g
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420617.png)

**2.创建oracle11g容器**

```bash
docker run -d -p 1521:1521 --name oracle11g registry.cn-hangzhou.aliyuncs.com/helowin/oracle_11g
```

**3.查看oracle11g容器是否创建成功**

```bash
docker ps -a
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420618.png)

**4.启动oracle11g容器**

```bash
docker start oracle11g
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420619.png)

**5.进入oracle11g容器进行配置**

```bash
docker exec -it oracle11g bash
```

**6.切换到root用户下进行配置**

```bash
su root
```

密码为：`helowin`

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420720.png)

**7、编辑profile文件配置ORACLE环境变量**

```bash
vi /etc/profile
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420721.png)

**8、最后添加以下3行配置**

```bash
export ORACLE_HOME=/home/oracle/app/oracle/product/11.2.0/dbhome_2
export ORACLE_SID=helowin
export PATH=$ORACLE_HOME/bin:$PATH
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420722.png)

保存 ：`:wq`
让配置生效：`source /etc/profile`

**9、创建软连接**

```bash
ln -s $ORACLE_HOME/bin/sqlplus /usr/bin
```

**10、切换到oracle 用户**

```bash
su - oracle
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420723.png)

**11、登录sqlplus并以 DBA 的身份登陆**

```bash
sqlplus /nolog
conn /as sysdba
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420724.png)

**12、修改sys、system用户密码并刷新权限**

```bash
alter user system identified by oracle;
alter user sys identified by oracle;
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420725.png)

退出：`exit;`
**13、查看一下oracle实例状态**

```bash
lsnrctl status
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420726.png)

**14、用nacivat连接oracle数据库**

服务名：`helowin`（一定要填写helowin）
密码：oracle（第12步设置的密码）

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420727.png)

## 基础概念和命令行

Oracle Database Management System（数据库管理系统）由 Database 和 Instance 构成：

- 在任何时刻，一个 Database 可以被多个 Instance 访问
- 而一个 Instance 只能连接一个 Database

存储结构：

- **逻辑存储结构：数据库 -> 表空间 -> 段 -> 区 -> Oracle 数据块**

- 物理存储结构：OS 文件 -> OS 块

用户：

- 用户是在某个 Instance 下被创建的，用户需要通过 Instance 里面的 table space 来连接数据库
- 用户连接后，对应的不是整个 database，而是 table space（表空间）。在 table space 里面，才能创建表等。
- 用户和 table space，是多对多的关系

---

切换到 root 用户后，进入 Oracle 路径：

```bash
su root -- 然后输入密码，这里是 helowin
su - oracle
```

使用 sqlplus 客户端：

```bash
sqlplus /nolog
```

登陆用户名为 sys，密码为 root 的 DBA 账号：

```bash
conn sys/root as sysdba
```

查看当前所有的实例（Instance）：

```bash
select instance_name from v$instance;
```

创建名为 test_space 的表空间（table space），文件的物理路径为 `/tmp/test_space.dbf` ，大小为 200M：

```bash
create tablespace test_space datafile '/tmp/test_space.dbf' size 200m;
```

创建用户名为 first_user，密码为 firstpwd 的用户，并设置其默认的表空间为 test_space;

```bash
create user first_user identified by firstpwd default tablespace test_space;
```

将 dba 的权限，赋予 first_user 用户（只有这样，才能保证该用户可以连接登陆）：

```bash
grant dba to first_user;
```

使用 first_user 为用户名，firstpwd 为密码，连接 helowin 实例（helowin 是一个自带的实例名）：

```bash
conn first_user/firstpwd@helowin
```













