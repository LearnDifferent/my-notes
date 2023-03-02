# Oracle 学习笔记

## 基础知识

### 安装部署

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

### 基础概念

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

伪表：

- MySQL 如果要查询系统当前时间，使用 `select now()` 就可以了，不需要 `from`
- 但是因为 Oracle 的 `select` 语句一定要加 `from` ，所以就有了伪表 `dual` 的概念
- 在 Oracle 中，查询当前系统时间，就使用 `select sysdate from dual;`

### 基础命令行

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

```sql
select instance_name from v$instance;
```

创建名为 test_space 的表空间（table space），文件的物理路径为 `/tmp/test_space.dbf` ，大小为 200M：

```sql
create tablespace test_space datafile '/tmp/test_space.dbf' size 200m;
```

创建用户名为 first_user，密码为 firstpwd 的用户，并设置其默认的 table space（表空间）为 test_space;

```sql
create user first_user identified by firstpwd default tablespace test_space;
```

如果要删除名为 test_space 的 table space（表空间），就使用：

```sql
drop tablespace test_space;
```

将 dba 的权限，赋予 first_user 用户（只有这样，才能保证该用户可以连接登陆）：

```sql
grant dba to first_user;
```

使用 first_user 为用户名，firstpwd 为密码，连接 helowin 实例（helowin 是一个自带的实例名）：

```sql
conn first_user/firstpwd@helowin
```

查看当前用户：

```sql
show user;
```

查看当前表空间里面的所有表（tables）：
```sql
select * from tab;
```

## DDL

> DDL: Data Definition Languages

查看一个 table 的字段及其数据结构，可以使用 `desc 表名;`。

查看所有表 / 视图是 `select * from tab;`

查找某个表 / 视图是 `select * from tab where tname like '%关键字%';`

- 注意：Oracle 区分大小写，所以上面模糊查询的关键字要注意大小写。

只希望查询表的时候，使用 `select * from tab where tabtype = 'TABLE';`

### 数据类型

**varchar2**

varchar2：可变长度的字符串。可以存储 `1` 到 `4000` 字节（字符）的值。 

创建时，必须指定字符串最大长度：

```
varchar2(最大长度)
```

### 约束

> Table（表）里面的约束

约束的作用：

- 在数据库中，我们通过**约束**来对每个字段中的数据的合法性进行规范。

约束的分类：

- Primary Key 主键约束
  - 非空且不能重复
- Unique Key 唯一性约束 / 唯一键
  - 不能重复
  - 可以为 null，且可以有多个 null
- Not Null 非空约束
  - 不能为 null
- Check 检查约束
  - 自定义的检查规则。比如，某个取值应该在某个范围内。
  - MySQL 中有，但是不生效。MySQL 中用 set 类型实现该效果。Oracle 中没有 set 数据类型。
  - Oracle 中有，且生效。
- 默认值约束
- Foreign Key 外键约束

**Oracle 没有自增长约束（auto_increment），但是可以使用 `create sequence` 来实现序列**。

## DML

### Oracle 特殊 SQL（对比 MySQL）

Oracle 中没有 MySQL 的 `limit`，而是使用 `rownum` 来实现。这个 `rownum` 实际上就是从 1 开始的序号。

如果只想返回表 test1 中的前 2 条数据，就直接写：

```sql
select * from test1 where rownum <= 2;
```

这里需要注意，**`rownum` 的后面只能接 `<` 或 `<=`**。（其实可以用 `where rownum = 1` 来返回第 1 条数据，但是因为从 `where rownum = 2` 开始就没效果了，所以就按照规范来，只能小于或小于等于）。

查询当前系统时间（相当于 MySQL 的  `select now();`）：

```sql
select sysdate from dual;
```

当 test1 表中的 id 字段为 null 时，替换为 0（相当于 MySQL 的  `select ifnull(id, 0) from test1;`）：

```sql
select nvl(id, 0) from test1; -- nvl 表示 null value
```

## 参考资料

- [2.oracle和mysql在库表操作上的区别](https://www.bilibili.com/video/BV1M34y1E7bW) 注：有[合集](https://space.bilibili.com/518627864/channel/collectiondetail?sid=433580&ctype=0)
- [B站讲的最好的oracle数据库教程全集（2022最新版）从入门到精通 数据库实战精讲 错过必后悔（附配套资料-两天掌握oracle）](https://www.bilibili.com/video/BV1zY4y1874D)
- [2023最新最适合Java开发人员学习的Oracle教程，从入门到工作实操](https://www.bilibili.com/video/BV1x8411M7cU)
