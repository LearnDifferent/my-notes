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

---

在 IDEA 上使用：

1. Data Source 选择 Oracle
2. Host 填写对应的 host
3. Port 填写默认的端口 1521
4. SID 填写默认的服务吗 helowin
5. User 和 Password 填写用户名和密码
6. 点击 Test Connection 连接
7. 如果连接成功后，如果想要查看连接配置，可以鼠标右键后，点击 Properties。或者使用快捷键 command + i

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

`varchar2`：可变长度的字符串。可以存储 `1` 到 `4000` 字节（字符）的值。 

创建时，必须指定字符串最大字节（byte）长度：

```
varchar2(最大长度)

-- 等价于：
varchar2(最大长度 byte)
```

如果想指定最大字符（char）长度：

```
varchar2(最大长度 char)
```

---

**char**

char 存储固定长度的字符串。可以存储 `1` 到 `2000` 字节的字符串。

```
char(固定长度)
char(固定长度 byte)

-- 如果要指定 char，就写成：
char(固定长度 char)

-- 如果不指定长度，那么长度就是 1
```

如果存储的值的长度小于列中定义的最大长度，则Oracle将空格填充到最大长度的字符串。 

如果插入一个长度大于列的值，Oracle会返回一个错误。

---

**nchar** 和 **nvarchar2**

Oracle `nchar` 用于存储固定长度的 Unicode 字符数据。`nchar` 的字符集只能是 `AL16UTF16` 或 `UTF8`，在数据库创建时指定为国家字符集。

`nchar` 不能指定 byte（字节长度），只能使用 char（字符长度），且只能使用当前国家默认字符集。

`nvarchar2` 同理。

---

**number**

`NUMBER`：用于存储可能为负值或正值的数值

语法：

```
number(精度, 位数)
```

精度：precision 表示一个数字中的位数。 范围从 `1` 到 `38`。

尺度（位数）：scale 是数字中小数点右侧的位数。 范围从 `-84` 到 `127`。

>例如，数字`1234.56`的精度是`6`，尺度是`2`。所以要存储这个数字，需要定义为：`NUMBER(6,2)`。

> 精度和尺度都是十进制数字，可选。 
>
> 如果跳过精度和小数位，Oracle 使用数字的最大范围和精度。

例如，下面定义了一个可以存储数值的数字，其数值范围和精度都是最大的：

```sql
number
```

定义一个精度为 `p` 的整数，可以使用下面的形式：

```sql
number(p)

-- 等价于精度为 p，尺度为 0 的定点数
number(p,0)
```

<u>Oracle 允许规模（位数）为负数</u>。例如，下面的数字将数值四舍五入到数百。

```sql
number(5,-2)
-- 如果是 1856，插入后会得到 1900
-- 如果是 1846，插入后会得到 1800
-- 如果是 856，插入后会得到 900
-- 如果是 846，插入后会得到 800
```

如果在 `number(p, s)` 列中插入数字，并且数字超过精度 `p`，则Oracle将发出错误。

 但是，如果数量超过尺度 `s`，则Oracle将对该值进行四舍五入。

---

**float**

`float` 数据类型是 `number` 的子类型。 其主要目的是促进与 ANSI SQL `float`数据类型的兼容：

```
float(精度)
```

我们只能指定 `float` 的精度（precision），不能指定尺度（scale）：

- 最大精度是 `126`
- Oracle 数据库是从数据中解析尺度的。

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

### Sequence

下文来自 ChatGPT

> 问：请告诉我 Oracle 数据库的 Sequence 是怎么使用的？

Oracle 数据库的 Sequence 用于生成唯一的数字序列，这些数字通常用作表的主键。

以下是使用 Oracle 数据库 Sequence 的步骤：

1. 使用 CREATE SEQUENCE 语句创建一个 Sequence 对象，设置初始值、步长和最大值等属性。例如：

```sql
CREATE SEQUENCE seq_employee_id
  START WITH 1
  INCREMENT BY 1
  MAXVALUE 999999
  NOCYCLE
  NOCACHE;
```

2. 在需要使用 Sequence 的表中，创建一个 Number 类型的列，用于保存 Sequence 生成的序列值。例如：

```sql
CREATE TABLE employees (
   employee_id NUMBER(6),
   first_name VARCHAR2(20),
   last_name VARCHAR2(20),
   hire_date DATE,
   salary NUMBER(8,2)
);
```

3. 在 INSERT 语句中通过函数 NEXTVAL() 调用 Sequence，将生成的新值插入到表中的 Number 列中。例如：

```sql
INSERT INTO employees (employee_id, first_name, last_name, hire_date, salary) 
  VALUES(seq_employee_id.NEXTVAL, 'John', 'Doe', TO_DATE('01-JAN-2010','DD-MON-YYYY'), 5000);
```

4. 如果需要重置 Sequence 的当前值，可以使用 ALTER SEQUENCE 语句修改当前值。例如：

```sql
ALTER SEQUENCE seq_employee_id RESTART WITH 1;
```

需要注意的是，Sequence 对象是 Oracle 数据库范围内的全局对象，可以被多个表共享。因此，在使用 Sequence 时，需要谨慎考虑序列的起始值、步长、最大值等属性，以确保生成的序列值不会重复或越界。



> 问：可以举例说明一下吗？

是的，以下是使用 Oracle 数据库 Sequence 的一个简单例子：

假设我们有一个名为 customers 的表，需要在该表中插入新记录时，自动为每个新记录生成一个唯一的 customer_id 值。

1. 使用 CREATE SEQUENCE 语句创建一个 Sequence 对象：

```sql
CREATE SEQUENCE seq_customer_id
  START WITH 1001
  INCREMENT BY 1
  MAXVALUE 9999
  NOCYCLE
  NOCACHE;
```

该 Sequence 对象的起始值为 1001，步长为 1，最大值为 9999，不允许循环，且不缓存生成的序列值。

2. 在 customers 表中创建一个 Number 类型的列 customer_id，用于保存 Sequence 生成的唯一 ID 值：

```sql
CREATE TABLE customers (
   customer_id NUMBER(4),
   first_name VARCHAR2(20),
   last_name VARCHAR2(20),
   email VARCHAR2(50)
);
```

3. 在 INSERT 语句中通过函数 NEXTVAL() 调用 Sequence，将生成的新值插入到表中的 customer_id 列中：

```sql
INSERT INTO customers (customer_id, first_name, last_name, email) 
  VALUES(seq_customer_id.NEXTVAL, 'John', 'Doe', 'john.doe@example.com');
```

4. 执行 INSERT 语句后，可以使用 SELECT 语句查询生成的唯一 ID 值：

```sql
SELECT customer_id, first_name, last_name, email FROM customers;
```

该查询结果可能类似于：

```bash
CUSTOMER_ID   FIRST_NAME   LAST_NAME   EMAIL
---------------------------------------------
1001          John        Doe         john.doe@example.com
```

5. 如果需要重置 Sequence 的当前值，可以使用 ALTER SEQUENCE 语句修改当前值：

```

ALTER SEQUENCE seq_customer_id RESTART WITH 2001;
```

执行完该语句后，下一个 INSERT 语句将会生成 2001 的 customer_id 值。



> 问：请再详细讲一下语法和参数

好的，下面我详细讲解一下 Oracle 数据库 Sequence 的语法和参数：

1. CREATE SEQUENCE 语法：

```sql
CREATE SEQUENCE sequence_name
  [ INCREMENT BY increment_value ]
  [ START WITH start_value ]
  [ MAXVALUE max_value | NOMAXVALUE ]
  [ MINVALUE min_value | NOMINVALUE ]
  [ CYCLE | NOCYCLE ]
  [ CACHE cache_value | NOCACHE ]
  [ ORDER | NOORDER ];
```

其中：

- sequence_name：必填，Sequence 对象的名称。
- INCREMENT BY increment_value：可选，指定 Sequence 中的数字增量值，默认为 1。
- START WITH start_value：可选，指定 Sequence 生成数字序列的起始值，默认为 1。
- MAXVALUE max_value | NOMAXVALUE：可选，指定 Sequence 可以生成的最大值，默认为 10^27-1，也可以设置为 NOMAXVALUE 表示无最大值限制。
- MINVALUE min_value | NOMINVALUE：可选，指定 Sequence 可以生成的最小值，默认为 1，也可以设置为 NOMINVALUE 表示无最小值限制。
- CYCLE | NOCYCLE：可选，指定 Sequence 是否循环，默认为 NOCYCLE 不循环，也可以设置为 CYCLE 表示循环。
- CACHE cache_value | NOCACHE：可选，指定 Oracle 服务器缓存的 Sequence 值的数量，默认为 20，也可以设置为 NOCACHE 表示不缓存。
- ORDER | NOORDER：可选，指定 Sequence 生成数字序列的顺序，默认为 NOORDER 无顺序，也可以设置为 ORDER 表示按顺序生成。

2. NEXTVAL() 函数

在 INSERT 语句中，通过调用 NEXTVAL() 函数来获取 Sequence 生成的下一个数字值。NEXTVAL() 函数的语法为：

```sql
sequence_name.NEXTVAL
```

其中 sequence_name 表示创建 Sequence 对象时指定的 sequence_name。

3. CURRVAL 函数

在查询语句中，可以使用 CURRVAL 函数来获取 Sequence 当前的数字值。CURRVAL 函数的语法为：

```sql
sequence_name.CURRVAL
```

需要注意的是，在使用 CURRVAL 函数之前，必须先调用 NEXTVAL() 函数生成过 Sequence 的值，否则会抛出 ORA-08002 异常。

4. ALTER SEQUENCE 语法

使用 ALTER SEQUENCE 语句来修改 Sequence 对象的属性值，语法为：

```sql
ALTER SEQUENCE sequence_name
  [ INCREMENT BY increment_value ]
  [ START WITH start_value ]
  [ MAXVALUE max_value | NOMAXVALUE ]
  [ MINVALUE min_value | NOMINVALUE ]
  [ CYCLE | NOCYCLE ]
  [ CACHE cache_value | NOCACHE ];
```

其中修改的属性与 CREATE SEQUENCE 时设置的属性相同。在修改 Sequence 的当前值时，可以使用 RESTART WITH 关键字，语法为：

```sql
ALTER SEQUENCE sequence_name RESTART WITH start_value;
```

需要注意的是，RESTART WITH 语句将清除之前生成过的所有 Sequence 值，因此需要谨慎使用。



> 问：请问 Oracle 数据库中的 Sequence 可以在多张表中使用吗？

可以，一个 Sequence 在 Oracle 数据库中是一个独立的对象，可以被许多表使用。在创建表时，可以使用 SEQUENCE 通过 NEXTVAL 或 CURRVAL 获取唯一的序列号来填充表中的列。

例如，可以使用以下语句对多个表指定相同的 Sequence：

```sql
CREATE SEQUENCE seq_example START WITH 1 INCREMENT BY 1;

CREATE TABLE table1 ( 
   id NUMBER DEFAULT seq_example.NEXTVAL,
   name VARCHAR2(50),
   address VARCHAR2(100)
);

CREATE TABLE table2 ( 
   id NUMBER DEFAULT seq_example.NEXTVAL,
   description VARCHAR2(200),
   amount NUMBER
);
```

每次向这些表中插入一行时，Sequence 将从下一个值开始产生唯一的序列号并加以使用。

个人补充：

- <u>上面的 SQL 语句有问题</u>，下面会解答问题所在
- Sequence 就是一个自增的工具，在表插入的时候使用。不管是哪张表，只要插入的时候使用了这个 Sequence，这个 Sequence 都不会清零重置，而是会继续自增下去



> 问：上面这段在 Oracle 数据库中执行的时候，出现了 `[42000][984] ORA-00984: 列在此处不允许` 的异常。请问是怎么回事？

在 Oracle 数据库中，不能将 `DEFAULT` 与 `SEQUENCE` 结合使用，因为 `DEFAULT` 只能接受常数、表达式或 SQL 函数，因此您必须显式调用 `SEQUENCE` 函数来获取下一个值。

因此，您可以通过以下方式更改 `id` 列定义：

```sql
CREATE TABLE table1 (
                        id NUMBER,
                        name VARCHAR2(50),
                        address VARCHAR2(100)
);

CREATE SEQUENCE seq_example START WITH 1 INCREMENT BY 1;

INSERT INTO table1 (id, name, address)
  VALUES (seq_example.NEXTVAL, 'John Doe', '123 Main St');
```

或者，如果您希望 `id` 自动增加，可以使用触发器：

```sql
CREATE TABLE table1 (
                        id NUMBER,
                        name VARCHAR2(50),
                        address VARCHAR2(100)
);

CREATE SEQUENCE seq_example START WITH 1 INCREMENT BY 1;

CREATE OR REPLACE TRIGGER table1_trigger
BEFORE INSERT ON table1
FOR EACH ROW
BEGIN
   :NEW.id := seq_example.NEXTVAL;
END;

INSERT INTO table1 (name, address) VALUES ('John Doe', '123 Main St');
```

## DML

**Oracle 特殊 SQL（对比 MySQL）**

Oracle 中没有 MySQL 的 `limit`，而是使用 <u>`rownum` 虚拟列</u>来实现。这个 `rownum` 实际上就是每一行，从 1 开始的序号。

如果只想返回 test1 表中的前 2 条数据，就直接写：

```sql
select * from test1 where rownum <= 2;
```

> 注意：**`rownum` 的后面只能接 `<` 或 `<=`**。
>
> 其实可以用 `where rownum = 1` 来返回第 1 条数据，但是因为从 `where rownum = 2` 开始就没效果了，所以就按照规范来，只能小于或小于等于。

如果要查询 test1 表中，第 2 条之后的数据：

```sql
select * from (
  select rownum as rn, test1.* from test1
) t where rn > 2;
```

> 上面要注意，因为每个新数据都会有 `rownum` ，所以这里要使用别名 `rn` 来设置子查询的 `rownum as rn` 。

如果要查询 test1 表中，第 2 到 3 条数据：

```sql
select a.* from (
  select
  	test1.*,rownum rn
  from test1 
  where rownum <= 3
) a where a.rn >= 2;
```

这样做分页，效率最高。

---

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
- [oracletutorial.com](https://www.oracletutorial.com/)
- [易百教程](https://www.yiibai.com/oracle)
