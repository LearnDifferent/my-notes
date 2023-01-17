[toc]

MySQL 设置时区（还可以参考 [MySQL查看和修改时区time_zone](https://app.yinxiang.com/shard/s72/nl/16998849/ebf84bdf-f5e6-4b7a-b2ef-6e762e7b0b08/)）：

```sql
set global time_zone = '+8:00';
set time_zone = '+8:00';
flush privileges;
```

# 基础

MySQL 是 RDBMS（关系型数据库管理系统/Relational Database Management System），用于管理数据库。

DDL 数据库定义语言
DML 操作
DQL 查询
DCL 控制

操作数据库：
1. 创建数据库： create database if not exists 库名;
2. 删除数据库：drop database if exists 库名;
3. 使用数据库：use 库名;
4. 查看数据库： show databases;

引擎：

- MyISAM ：使用表锁，在高并发下会出现严重的问题，影响效率
- InnoDb：行锁 

## 数据库的列类型

数值：
- tinyint：1个字节
- smallint：2个字节
- mediumint：3个字节
- int：4个字节（常用）
- bigint：8个字节
- float：浮点数 4个字节（常用）
- double：浮点数 8个字节
- decimal：字符串形式的浮点数（用于金融计算）

字符串：
- char：0-255 的固定大小字符串
- varchar：0-65535 的可变字符串（常用）
- tinytext：2^8-1 的微型文本串
- text：2^16-1 的文本串（保存大文本）

时间日期：
- date：YYYY-MM-DD
- time：HH:mm:ss
- datetime：YYYY-MM-DD HH:mm:ss
- timestamp
- year

null：
- 没有值或未知
- 如果使用 null 运算，结果还是为 null，所以不要使用它来运算

## 数据库的字段属性（重点）

Unsigned：
- 无符号的整数
- 声明该列不能为负数

zerofill：
- 比如 int(3)，也就是 3 个长度的 int
- 如果填入 5 的话，就会变成 005

auto_increment：
- 用来设置唯一的自增整数类型主键/ID

allow null/not null：
- 是否允许不填值

Default：
- 设置默认值，不填的话就采用默认值

每个项目都建议有以下五个字段：
* id：主键
* version：乐观锁
* is_delete：伪删除
* gmt_create：创建时间
* gmt_update：修改时间

```sql
create table [if not exist] `表名` (
    `字段名` 列类型 [属性] [索引] [注释]
    .....
    primary key(`id`)
)engine=innodb default charset=utf8
-- 创建数据库基本语法

CREATE TABLE `tb1_user` (
`id` int(11) unsigned NOT NULL AUTO_INCREMENT,
`email` varchar(255) DEFAULT NULL,
`last_name` varchar(50) DEFAULT NULL,
`time` datetime(6) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
-- 复制了一份数据库创建语句

show create database [数据库名称]; -- 查看数据库创建语句

show create table [表名]; -- 显示表的结构

desc [表名] -- 显示表的结构
```

## 数据表的类型

引擎：
- 默认使用 INNODB：安全性高，支持事务处理，支持外键约束，可以多表多用户操作
- 早些年使用 MYISAM：节约空间（只有 INNODB 的一半），速度较快

## 修改删除表

```sql
alter table [旧表名] rename as [新表名];

alter table teacher add age int(11);
-- alter tabale [表名] add [字段名] [列属性];
-- 用于添加字段

alter table teacher modify age varchar(11);
-- 修改字段的约束，让 age 从 int 变为 varchar

alter table teacher change age age1;
-- 修改 age 字段的名称为 age1

alter table teacher drop age1;
-- 删除 age1 字段

drop table if exists teacher; -- 删除表
```

## 表相关操作的补充

查看列：desc 表名;
修改表名：alter table t_book rename to bbb;
添加列：alter table 表名 add column 列名 varchar(30);
删除列：alter table 表名 drop column 列名;
修改列名MySQL： alter table bbb change nnnnn hh int;
修改列名SQLServer：exec sp_rename't_student.name','nn','column';
修改列名Oracle：lter table bbb rename column nnnnn to hh int;
修改列属性：alter table t_book modify name varchar(22);

# MySQL 数据管理
## 外键

方式一：创建表的时候，添加外键约束（constraint）。

方式二：创建表成功后，再添加外键约束

```sql
alter table [表名] add constraint `fk_约束名` foreign key（`作为外键的列`） references `引用的表`(`引用的字段`)

alter table `student` add constraint `fk_gradeid` references `grade`(id);
-- 让 student 表的 gradeid 字段，引用到 grade 表的 id 字段
```

删除有外键关系的表的时候，必须先删除从表，再删除主表。

以上的操作都是物理外键，是数据库级别的外键，可能会造成数据库过多的问题，所以了解即可。

推荐：
- 数据库是单纯的表，只用来存储数据，只有行（数据）和列（字段）
- 我们想要使用多张表的数据，想要使用外键的时候，应该使用程序去实现

## DML 语言（重点）

插入语句：

```sql
insert into `stu`(`name`) values ('张三');

insert into `grade`(`gradename`) values ('大一'), (`大二`);

insert into `stu`(`name`, `gender`) values ('张三', '男');

insert into `stu` values ('张三', '男');

 insert into `stu`(`name`, `gender`) 
 values ('李四', '女'), ('王五', '男');
```

修改语句：

```sql
update `stu` set `name`='王五' where id = 1;

update `stu` set `name`='王五', `email`='12f@qq.com' where id = 1;

update `stu` set `name`='前100名' where id <= 100;

update `stu` set `name`='除了第一名' where id != 1;

update `stu` set `name`='第二名到第五名（包括第二名和第五名）' where id between 2 and 5;

update `stu` set `name`='王五' where `name`='李四' and `gender`='女';

update `stu` set `name`='王五' where `name`='李四' or `gender`='女';

update `stu` set `createDate`=current_time where id = 1; -- current_time 是一个变量，代表当前时间
```

删除语句：

```sql
delete from stu where id = 1;

truncate table `stu`; -- 清空表里面的数据，不改变表的结构和索引约束等。

/*
truncate 会重新设置自增列，也就是让 ID 可以从零开始重新排序

truncate 不会影响事务
*/ 
```

# DQL 查询数据（最重要！单独一个标题）

Data Query Language/数据查询语言

## 基础查询
```sql
select * from student;

select studentNo as 学号, studentName as 姓名 from student; -- 可以起别名

-- 数据库的列还可以用表达式形式

select concat('姓名：', studentName) as '姓名+名字的格式' from student; -- concat 函数可以拼接字符串
/*
姓名+名字的格式
姓名：张三
姓名：李四
*/

select distinct studentNo from result; -- 使用了去重操作，剔除了重复的数据。重复的数据只显示一条
/*
我们要查询所有有考试成绩的学生的学号，也就是哪些学生参加了考试
因为一个学生可能有很多科目的数据，如果不使用 distinct 直接查学号，就会查出来很多学号
*/

select version(); -- 查询 mysql 版本号
select 100*5-1; -- 计算数字
select @@AUTO_INCREMENT_increment; -- 查询自增的步长

select `studentNo` , `studentResult` + 1 as '所有成绩+1分' from result -- 给 studentResult 的结果的数值 +1

-- select [表达式] from [表];
```

## 模糊查询

```sql
select subjectno, subjectname from subject where subjectname like 'java%';
-- 查询以 java 为开头的课程（只在后面用了百分号%，所以前面必须完全匹配，而后面随意）

select subjectno, subjectname from subject where subjectname like '%a%';
-- 查询中间含有「a」名称的课程

select gradename from grade where gradename like '大_'; 
-- 查询「大」+ 1个字符的年级名称（这里只用了一个下划线）

select studentNo, studentName from student where studentName like '周__';
-- 查询「周」+ 2个字符的学生名称（这里用了两个下划线，所以能查出「周某某」，而不能查出「周某」）

select subjectNo, subjectname from subject where subjectNo in (6,10,15);
-- 查询代号为 6,10 和 15 的课程。注意！in 关键字是制定具体值的，不能使用 like 的下划线或百分号

select studentName from student where email='';
-- 查询邮箱为空字符串的学生（注意，这里不是 null，而是空字符串）

select studentName from student where gradeid is null;
-- 查询年级 id 为 null 的学生。（注意，这里是 null！对于 null 应该使用 is null 关键字）

select studentName from student where email='' or email is null;
-- 如果不知道「空」代表空字符串还是 null，就用 or 串联两个判断

select studentName from student where gradeid is not null;
-- 如果判断的条件是「不为 null」，就使用关键字 is not null
```

| 比较运算符 | 语法 | 描述 |
| --- | --- | --- |
| is null | a is null | 如果 a 是 null，为 true |
| is not null | a is not null | 如果 a 不是 null，为 true |
| **like** | a like b | 如果 a 匹配 b，则为 true|
| **in** | a in (a1,a2,a3) | 假设 a 是 a1 或 a2 或 a3 之中的 1 个值，就为 true |

## 联表查询

```sql
/*
student 和 result 中都有 studentNo 的字段
，我们通过 studentNo 来查询两个表的数据
，注意，studentNo 要指定为其中一个表的 studentNo
，要不就会模棱两可。

注意：下面会用到的 join [连接的表] on 
是「连接查询」，其中的 on 用于判断条件。
而 where 是等值查询。他们的效果是一样的。
*/
select s.studentNo, studentName, SubjectNo, SutdentResult
from student as a
INNER join result as r
on s.studentNO = r.studentNo;
-- inner join：如果表中至少有一个匹配，就返回全部的结果，也就是并集的意思

select s.studentNo, studentName, SubjectNo, SutdentResult
from student as a
LEFT join result as r
on s.studentNO = r.studentNo;
-- left join：左边的表（student）所有值都会返回，而右表（result）中只会返回匹配的值

select s.studentNo, studentName, SubjectNo, SutdentResult
from student a
RIGHT join result r
on s.studentNO = r.studentNo;
-- right join：右表（result）里所有的值会被返回，而左表（student）只有匹配的值会被返回

select s.studentNo, studentName, SubjectNo, SutdentResult
from student as a
LEFT join result as r
on s.studentNO = r.studentNo
where studentResult is null;
/*
查询缺考的学生：
因为使用了 left join
，所以会得到左表的 student 的全部信息
，通过 studentNo 来获得右表 result 的信息
，然后根据 result 中的 studentResult 为 null
，筛选出缺考的学生
*/
```

如果要查询参加了考试的同学的学号，学生姓名，科目名和分数：
- 这里用到了 student 表（学号和姓名），result 表（分数）和 subject 表（科目名）
- 我们可以通过 student 表里面的 studentNo 来连接 result 表的 studentNo
- 然后再通过 result 表里的 subjectNo 来连接 subject 表里的 subjectNo。
- 因为查询的是参加了考试的同学，所以筛选的关键是 studentResult 不能为 null
- 也就是说，一开始需要完整显示的表是 result
- 即，使用 student right join result 得出一个表
- 新得出的表里面有 result 的 subjectNo 信息
- 用该 subjectNo 连接 subject 表，从而得出科目名
- 此时使用 inner join 即可（因为只要条件符合，就会得出结果）

```sql
select stu.studentNo, studentName, subjectName, studentResult 
from student as stu right join result as r 
on stu.studentNo = r.studentNo 
inner join `subject` as sub
on r.subjectNo = sub.subjectNo;
```

## 子查询（了解）

使用括号进行由里到外的嵌套查询，类似于联表查询（推荐使用联表查询）：

```sql
/*
查询“高等数学”的分数大于等于 80 的学生的学号和姓名。
student 表里面有 studentNo 和 studenetName
result 表里面有 studentNo，studentResult 和 subjectNo
subject 表里面有 subjectNo 和 subjectName
*/

-- 这里只连接 student 和 result，然后使用子查询嵌套查询 subject 的结果。
-- 也就是说，先查出名字为‘高等数学’的 subjectNo
-- 然后利用查出来的 subjectNo 作为判断条件传入联合的表中

select s.studentNo, studentName
from student s
inner join result r
on s.studentNo = r.studentNo
where studentResult >= 80
and subjectNo = (
    select subjectNo from subject where subjectName = '高等数学'
);
```

## 自连接（了解）

原来的表（表名为 dalaodi）：

| id | pid | name |
| --- | --- | --- |
| 2 | 1 | 大佬A |
| 3 | 1 | 大佬B |
| 4 | 1 | 大佬C |
| 5 | 2 | 小弟A |
| 6 | 3 | 小弟B |
| 7 | 4 | 小弟C |
| 8 | 4 | 小弟D |

父类的表（pid 为 1 代表是大佬）：

| id | name |
| --- | --- |
| 2 | 大佬A |
| 3 | 大佬B |
| 4 | 大佬C |

子类的表（小弟的 pid 对应大佬的 id）：

| id | pid | name |
| --- | --- | --- |
| 5 | 2 | 小弟A |
| 6 | 3 | 小弟B |
| 7 | 4 | 小弟C |
| 8 | 4 | 小弟D |

我们希望得到的「父类对应的子类的关系表」（大佬和小弟关系表）：

| 大佬名 | 小弟名 |
| --- | --- |
| 大佬A | 小弟A |
| 大佬B | 小弟B |
| 大佬C | 小弟C |
| 大佬C | 小弟D |

SQL：

```sql
-- 把一张表，看成两张一样的表，区分为 a 和 b 表
select a.`name` as '大佬名', b.`name` as '小弟名'
from `dalaodi` as a, `dalaodi` as b
where a.id = b.pid;
-- 查询 id 和 pid 相等的数据，栏目名为 大佬名 和 小弟名
```

## 分页和排序

例子 - 按照学生成绩排序：

```sql
select `studentResult` as '成绩', `studentName` as '姓名' 
from student s left join result r 
on s.studentNo = r.studentNo
order by studentResult desc -- desc 是从大到小，asc 反之
```

分页关键字是 `limit`：
- limit 0, 5：从第 1 条数据开始，每页显示 5 条数据
- limit 1, 4：从第 2 条数据开始，每页显示 4 条数据
- limit 5, 5：从第 6 条数据开始，每页显示 5 条数据，也就是「从第 2 页的第 1 条数据开始」
- limit 6, 5：从第 7 条数据开始，每页显示 5 条数据，也就是「从第 2 页的第 2 条数据开始」

```sql
select `studentResult` as '成绩', `studentName` as '姓名' 
from student s left join result r 
on s.studentNo = r.studentNo
order by studentResult desc
limit 5,5 -- 从第 6 条数据开始显示 5 条数据
-- 也就是从第 2 页的第 1 条开始（每页显示 5 条数据）
-- 注意！limit 的关键字要在最后面，也就是不能早于 order by
```

## MySQL 函数

### MySQL 常用函数（并不常用）

```sql
select user(); -- 获取当前 MySql 用户信息

-- 需要记住的是「日期和时间的函数」：
select current_date(); -- YYYY-MM-dd

select now() -- YYYY-MM-dd HH:mm:ss

select localtime() -- 获取本地时间 YYYY-MM-dd HH:mm:ss

select sysdate() -- 获取系统时间 YYYY-MM-dd HH:mm:ss

select year(now()); -- 获取当前的年份

-- 数学运算：
select abs(-8); -- 8  绝对值

select ceiling(9.4) -- 10  向上取整

select floor(9.4) -- 9  向下取整

select rand() -- 返回 0 到 1 之间到随机数

select sign(0) -- 判断一个数的符号，sign(0) 得到 0
select sign(-10) -- 得到 -1
select sign(10) -- 得到 1

-- 字符串函数：
select char_length('四个字符') -- 4  获得字符串长度

select concat('o','m','g') -- omg  拼接字符串

select lower('ToLower') -- tolower
select upper('ToUpper') -- TOUPPER

select replace('测试一下', '测试', 'test') -- test一下

-- 把姓「王」的同学改为「黄」
select replace(studentName, '王', '黄')
from student where studentName like '王%';
```

### 聚合函数（常用）

```sql
-- 查询表中有多少个记录：
select count(studentName) from student; -- count(字段) 会忽略所有的 null 值，速度可能最快

select count(*) from student; -- count(*) 用于计算行数，会把所有的列都走一遍，所以可能会慢（不会忽略 null 值）

select count(1) from student; -- count(1)  计算某 1 列的行树，比 count(*) 快一点（不会忽略 null 值）


select sum(studentResult) as 总和 from result;

select avg(studentResult) as 平均分 from result;

select max(studentResult) as 最高分 from result;

select min(studentResult) as 最低分 from result;

```

## 分组和分组后过滤条件

例子 - 根据不同的课程分组，查询每个课程的平均分，最高分和最低分。然后使用过滤条件：平均分要大于 80

其中 result 表内有 subjectNo 和 studentResult
，subject 表内有 subjectNo 和 subjectName：

```sql
select subjectName, avg(studentResult) as 平均分, max(studentResult), min(studentResult)
from result r
inner join `subject` s
on r.subjectNo = s.subjectNo
group by s.subjectName -- 根据 subjectName 来分组
having 平均分 > 80; -- 然后要求平均分大于 80
```

## Window Function 窗口函数

**窗口函数表达式**：

```sql
function (args) over ([partition by 分组条件]  [order by 排序字段 [desc] ]  [frame] )
```

例如：

- `row_number() over (order by id)` 不分组，直接按照 id 升序
- `rank() over (partition by gender order by gender desc)` 根据 gender 排序后，再根据 gender 降序

### 窗口函数 + 排序函数

- `row_number()` 
  - 形如：1, 2, 3 ...
  - 序号不重复，且连续
- `rank()` 
  - 形如：1, 2, 2, 4, 5
  - 序号可以重复（并列），且不连续
- `dense_rank()`
  - 形如：1, 2, 2, 3, 3, 4
  - 序号可以重复（并列），且连续

例如：

Q：*表 Scores 的列为 id 和 score，score 表示分数。按照分数从高到低排名，如果多个分数相同，就并列名次。排名应该是连续的整数，也就是排名之间不能有空缺的数字。*（LeetCode 178）

A：

```sql
select score, dense_rank over(order by score desc) as `rank` from Scores
```

---

Q：*表 Employee 有 id, name, salary, departmentId，表 Department 有 id 和 name。其中 Employee 表的 departmentId 对应 Department 表的 id。找到每个部门薪资最高的员工（有多个时，返回多个员工），按任意顺序返回结果。*（LeetCode 184）

A：

```sql
select Department, Employee, salary from
from (
    select
        d.name as Department, -- 部门名
        e.name as Employee, -- 员工名
        salary, -- 工资
    	-- rank()：允许多个并列。这里也可以使用 dense_rank()
    	-- partition by departmentId：根据部门来分组
    	-- order by salary desc：根据工资来排序
        rank() over(partition by departmentId order by salary desc) as salary_rank
    from
        Employee e
    inner join
        Department d on e.departmentId = d.id
) as t -- 记得要给表一个别名

where
    salary_rank = 1; -- 只返回 按照部门分组后，每个部门薪资排名第 1 的员工
```

### 窗口函数 + 聚合函数

一般使用 aggregate function（聚合函数）的时候，需要使用 group by 来聚合。

但是如果加上了  `over()` 就不需要再聚合了。如下，列出 每个日期、每个日期的订票金额 和 总的订票数：

```sql
select booking_date, amount_tipped, sum(amount_tipped) over() from bookings;
```

这就实现了 run an aggregate function without aggregating any values。

> The aggregation was performed internally behind the scenes and then the value was added for each individual entry in our table

We can use aggregate functions with such window functions in a different way. So we can use the capabilities of the aggregate function, but without the aggregation of the entire table.

SQL 执行结果类似于：

| booking_date | amount_tipped | sum(amount_tipped) |
| ------------ | ------------- | ------------------ |
| 2022-01-01   | 4.1           | 25.0               |
| 2022-01-01   | 5.9           | 25.0               |
| 2022-01-02   | 2.2           | 25.0               |
| 2022-01-02   | 2.8           | 25.0               |
| 2022-01-03   | 10.0          | 25.0               |

---

现在加深一下对窗口函数的理解，在上面的 SQL 语句的基础上添加 `partition by booking_date` ，也就是按照订票日期来分组：

```sql
select booking_date, amount_tipped, sum(amount_tipped) over(partition by booking_date) from bookings;
```

This means we want to check if we have any equals values in the `booking_date` column, and all equal values will be the sum of the amount `amount_tipped` calculation.

So for two equal dates so to say, the sum will be calculated based on these two dates.

> This again, happens behind the scenes, so we don't actually reduce any input fields or any input values.

SQL 执行结果类似于：

| booking_date | amount_tipped | sum(amount_tipped) |
| ------------ | ------------- | ------------------ |
| 2022-01-01   | 4.1           | 10.0               |
| 2022-01-01   | 5.9           | 10.0               |
| 2022-01-02   | 2.2           | 5.0                |
| 2022-01-02   | 2.8           | 5.0                |
| 2022-01-03   | 10.0          | 10.0               |

也就是说，如果没加 `partition by` ，`sum()` 就是求出所有行的总和，并放到每一行中。

当加上 `partition by` 后，`sum()` 就是先分组，然后求出每个分组的行的总和，并放到每个分组的行上。

> The window function doesn't aggregate or group the data in the table.
>
> It just applies this calculation behind the scenes to retrieve the sum for these equal values here.

## 数据库级别的 MD5 加密

MD5 不可逆向破解

```sql
-- 加密当前存在的密码（不推荐这样做）
update tab_md5 set password=md5(password) where id=1;
update tab_md5 set password=md5(password);

-- 推荐在插入的时候加密
inert into tab_md5 values(1,'username',md5('123456'));

-- 校验的时候，将用户传递进来的密码，先加密，然后再和数据库中的加密值比对
select * from tab_md5 where name='username' and password=md5('123456');
```

在 Spring Boot 中可以使用 DigestUtils.md5DigestAsHex() 来传入 byte 数组，将其转化为 MD5 加密后的字符串。

## 查询语法的顺序

select 语法顺序：

1. `distinct`：去重复
2. (表名.)字段 (`as`) 别称（或者星号 \* 代表全部字段）
3. `from` 表名 (`as`) 别称
4. `inner join`（或者 left / right) 另一个表名 `on` 等值判断
5. `where` 筛选条件
6. `group by` 字段：按照该字段分组
7. `having` 条件：分组后根据条件再进行过滤
8. `order by` 字段 `desc`：根据该字段降序排序（asc 从小到大排序）
9. `limit 1, 5`：从第 2 条数据开始的 5 条数据

# 业务级别的 MySQL 学习

## 事务

ACID 数据库事务正确执行的是个基本要素：

* Atomicity：原子性，两个动作，要么一起成功，要么一起失败
* Consistency：最终一致性，最后的总价值肯定不变
* Isolation：隔离性，多个用户同时操作，要排除其他业务对当前业务的影响
* Durability：持久性，只要事务没有提交，就恢复原状；只要事务提交了，才持久化到数据库

事务的隔离级别/隔离所导致的问题：

* 脏读：一个事务读取了另一个事务还未提交的数据
* 不可重复读：读取表中某行数据时，多次读取的结果不同（可能因为刚好数据更新了，所以不一定是错误）
* 虚读/幻读：读取到了别的事务插入的数据

## 索引

> 索引/Index 帮助 MySQL 高效获取数据的数据结构

索引的分类：
- Primary Key/主键索引：
    - 唯一的标识，主键不可重复
    - 只能有一个列作为主键
- Unique Key/唯一索引：
    - 避免重复的列
    - Unique Key 可以有多个，多个列都可以设置为 Unique Key
- Key/Index/常规索引：
    - 默认使用 index 关键字或 key 关键字来设置
- FullText/全文索引：
    - 只有支持 FullText 的数据库内才有（MyISAM 支持）
    - 用于快速定位数据
    

```sql
show index from student; -- 显示所有的索引

alter table school.student add fulltext index name(studentName);
-- 在 school 数据库下的 student 表内，添加一个 全文索引，索引的名字为 name，对应列名为 studentName

/*
分析 sql 执行的状况

看 row，也就是查询了多少条记录得出结果

如果使用了索引，row 的数量就是 1，代表查一次就查出来了
*/
explain select * from student; -- 适用于非全文索引

explain select * from student where match(name) against('刘'); -- 适用于全文索引

/*
添加索引的方式：

索引名格式：id_表名_字段名
创建格式：create index 索引名 on 表(字段)

如果是 fulltext 就是：create fulltext index 索引名 on 表(字段)
*/
create index id_app_user_name on app_user(name);
```

索引相当于唯一定位，同样是执行 select 语句，用了索引后，速度就会快非常多。

索引原则：
1. 不要对经常改动的数据添加索引
2. 数据量小的表不需要添加索引
3. 索引一般加在，需要经常被用来查询的字段上

## 用户权限管理

```sql
-- 创建用户名为 iamuser，密码为 123456 的用户：
create user iamuser identity by '123456';

-- 修改当前用户密码：
set password = password('654321');

-- 修改指定用户密码 / 修改 anotheruser 的密码为 111111：
set password for anotheruser = password('111111');

-- 重命名 / 修改用户 iamuser 的名字为 newuser：
rename user iamuser to newuser;

-- 将在所有库.所有表的 所有的权限 授权给 newuser（除了给别人授权的权限）：
grant all privileges on *.* to newuser;
-- 注意，只有 root 用户可以给别人授权

-- 查询指定用户的权限：
show grants for newuser;
-- 查询 root 用户的权限：
show grants for root@localhost;

-- 撤销权限：
revoke all privileges on *.* from newuser;

-- 删除用户：
drop user newuser;
```

## 备份数据库

mysqldump -h主机 -u用户名 -p密码 数据库名称 表名 > 导入的磁盘位置/名称.sql：
`mysqldump -hlocalhost -uroot -prootroot school student > ~/Desktop/stu.sql`
注意：如果怕密码暴露，`-p` 后面可以为空，然后命令行会让你输入密码

dump 多张表：
`mysqldump -hlocalhost -uroot -prootroot school student grade > ~/Desktop/stuAndGra.sql`

dump 某个数据库（这里的数据库名称是 school）：
`mysqldump -hlocalhost -uroot -prootroot school > ~/Desktop/db_school.sql`

如果要导入数据库，先登陆 mysql，然后使用 `use 数据库名称;` 切换到相应数据库，之后：
`source ~/Desktop/stu.sql` （source 文件目录/文件名.sql）

## 设计数据库

软件开发中，设计数据库：

1. 分析需求：分析业务和需要处理的数据库的需求
2. 概要设计：设计关系图 E-R 图

设计个人博客的数据库：

1. 收集信息，分析需求
    1. 用户表：登陆注销，用户个人信息，写博客，创建文类
    2. 分类表：文章分类，作者
    3. 文章表：文章信息
    4. 友情链接
    5. 自定义表：系统信息、某个关键字、一些主字段（主要是不常用，直接搞个数据库定义一些 key: value 就可以了）
2. 标识实体：把需求落地到每个字段
3. 标识实体之间的关系：
    1. 写博客：user -> blog
    2. 创建文类 user -> category
    3. 关注关系：user -> user
    4. 评论： user -> user -> blog
    5. 友情链接：links


数据库设计的三大范式：

1. 1NF / 第一范式：保证原子性，每一个字段都不可以再细分了
2. 2NF / 第二范式：前提是满足 1NF，让每张表只描述一件事情，如果描述了两件事情就拆分
3. 3NF / 第三范式：前提是满足 1NF 和 2NF，让每一列的数据都和主键直接相关

规范性和性能要权衡，以性能优先，关联查询的表要在 3 张之内。
实际上设计表的时候，要故意留出冗余的字段。
在数据量大的时候，可以增加计算列，也就是增加数据的时候，这个计算列也 +1，需要查询总数的时候，就直接查询计算列即可，这样比索引还好一点。
