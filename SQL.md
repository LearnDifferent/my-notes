# Window Function 窗口函数

**窗口函数表达式**：

```sql
function (args) over ([partition by 分组条件]  [order by 排序字段 [desc] ]  [rows / range] )
```

`over()` clause:

- `partition by` divides the result into partitions
- `order by` defines the logical order of the rows within each partition of the result
- `rows / range` limits the rows within a partition by specifiying the starting and end points of a window frame

例如：

- `row_number() over (order by id)` 不分组，直接按照 id 升序
- `rank() over (partition by gender order by gender desc)` 根据 gender 分组后，再根据 gender 降序

## 窗口函数 + 排序函数

- `row_number()` 
  - 形如：1, 2, 3 ...
  - 序号不重复，且连续
- `rank()` 
  - 形如：1, 2, 2, 4, 5
  - 序号可以重复（并列），且不连续
- `dense_rank()`
  - 形如：1, 2, 2, 3, 3, 4
  - 序号可以重复（并列），且连续

`partition by` 可以参考[下文](#rank_window_func)。

例如：

Q：*表 Scores 的列为 id 和 score，score 表示分数。按照分数从高到低排名，如果多个分数相同，就并列名次。排名应该是连续的整数，也就是排名之间不能有空缺的数字。*（LeetCode 178）

A：

```sql
select score, dense_rank() over(order by score desc) as `rank` from Scores
```

---

Q：*表 Employee 有 id, name, salary, departmentId，表 Department 有 id 和 name。其中 Employee 表的 departmentId 对应 Department 表的 id。找到每个部门薪资最高的员工（有多个时，返回多个员工），按任意顺序返回结果。*（LeetCode 184）

<span id='rank_window_answer'>A：</span>

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

## 窗口函数 + 聚合函数

一般使用 aggregate function（聚合函数）的时候，需要使用 group by 来聚合。

但是如果加上了  `over()` 就不需要再聚合了。如下，列出 每个日期、每个日期的小费金额 和 总的小费金额：

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

可以这样理解，**如果没有 `partition by` ，就是所有数据在一个窗口中进行计算。而 `partition by` 就是在每个分组的窗口中进行计算**。

---

在上面的基础上，如果加上了 `order by` ：

下面的 SQL 中，booking_date 表示 预定日期，amount_tipped 表示 小费金额，amount_billed 表示 金额（这里假设数据里面的 amount_billed 和 amount_tipped 的大小相等）：

```sql
select booking_date, amount_tipped, sum(amount_tipped) over(partition by booking_date order by amount_billed) as sum_added from bookings;
```

SQL 执行结果类似于：

| booking_date | amount_tipped | sum_added |
| ------------ | ------------- | --------- |
| 2022-01-01   | 4.1           | 4.1       |
| 2022-01-01   | 5.9           | 10.0      |
| 2022-01-02   | 2.2           | 2.2       |
| 2022-01-02   | 2.8           | 5.0       |
| 2022-01-03   | 10.0          | 10.0      |

当 **aggregation function** 遇上有 `partition by` 和 `order by` 的 **window function** 时，会在一个分组内，根据 `order by` 的顺序，**计算每一行的累加和**（如上所示）。

> 实际上，这是因为**加上了 `order by` 之后，会在 window 的基础上，产生下文提到的 [Frame](#Frame_Clause)**。
>
> 对于 Frame 而言，其默认的 frame clause 就是将“当前分组的第 1 行数据，到当前行”，作为 1 个 frame 来处理。
>
> 也就是说，此时的聚合函数，计算的范围不是当前分组，而是当前分组再细分的一个范围。

---

如果不使用聚合函数，而是使用排序函数的话：

```sql
select booking_date, amount_tipped, row_number() over(partition by booking_date order by amount_tipped desc) as ranking from bookings;
```

<span id='rank_window_func'> 此时的 SQL 表示按照 booking_date 为分组，每个分组内，根据 amount_tipped 降序 排序。</span>

SQL 执行结果类似于：

| booking_date | amount_tipped | ranking |
| ------------ | ------------- | ------- |
| 2022-01-01   | 5.9           | 1       |
| 2022-01-01   | 4.1           | 2       |
| 2022-01-02   | 2.8           | 1       |
| 2022-01-02   | 2.2           | 2       |
| 2022-01-03   | 10.0          | 1       |

[上文中，此答案](#rank_window_answer)的子查询就是这个道理。

## Analytic Functions

### lag() / lead()

`lag()` and `lead()` .

使用 `lag()` 可以 access the rows before the current row（获取当前行 前面的值）。

>**`lead()` 就是 access the rows after the current row（获取当前行 后面的的数据）**。用法可以参考 `lag()` 。

比如，按照部门分组、根据 ID 排序，获取每个分组中，在当前 ID 的职员信息那一行中，放入前一个 ID 的职员的工资：

```sql
select 
	emp_id, 
	emp_name, 
	dept_name, 
	salary, 
	lag(salary) over(partition by dept_name order by emp_id) as prev_emp_salary
from employee_table;
```

SQL 执行结果类似于：

| emp_id | emp_name | dept_name | salary | prev_emp_salary |
| ------ | -------- | --------- | ------ | --------------- |
| 1      | James    | Finance   | 4000   | null            |
| 2      | John     | Finance   | 5000   | 4000            |
| 3      | Sally    | HR        | 2000   | null            |
| 4      | Mary     | HR        | 3000   | 2000            |
| 5      | Peter    | HR        | 1500   | 3000            |

也就是获取每个窗口中，前一行相应位置的值。

`lag()` 函数可以带上参数，也就是 **`lag(scalar_expression, [offset], [default])`**：

- scalar_expression：指定需要的数据的 column
- offset：前面第几行，默认为 1，表示需要前面 1 行。offset can't be a negative value.
- default：set the default value to be used if a previous (or following) row does not exist，也就是 null 的替换值

比如说，需要前面第 2 行（上一行的上一行）的 salary 的值：

```sql
select 
	emp_id, 
	emp_name, 
	dept_name, 
	salary, 
	lag(salary, 2) over(partition by dept_name order by emp_id) as prev_emp_salary
from employee_table;
```

SQL 执行结果类似于：

| emp_id | emp_name | dept_name | salary | prev_emp_salary |
| ------ | -------- | --------- | ------ | --------------- |
| 1      | James    | Finance   | 4000   | null            |
| 2      | John     | Finance   | 5000   | null            |
| 3      | Sally    | Finance   | 2000   | 4000            |
| 4      | Mary     | Finance   | 3000   | 5000            |
| 5      | Peter    | Finance   | 1500   | 2000            |
| 6      | Tim      | HR        | 2000   | null            |
| 7      | Johnson  | HR        | 1000   | null            |
| 8      | Larry    | HR        | 1500   | 2000            |

再比如，需要前面第 2 行（上一行的上一行）的 salary 的值，如果是 null 就替换为 0：

```sql
select 
	emp_id, 
	emp_name, 
	dept_name, 
	salary, 
	lag(salary, 2, 0) over(partition by dept_name order by emp_id) as prev_emp_salary
from employee_table;
```

SQL 执行结果类似于：

| emp_id | emp_name | dept_name | salary | prev_emp_salary |
| ------ | -------- | --------- | ------ | --------------- |
| 1      | James    | Finance   | 4000   | 0               |
| 2      | John     | Finance   | 5000   | 0               |
| 3      | Sally    | Finance   | 2000   | 4000            |
| 4      | Mary     | Finance   | 3000   | 5000            |
| 5      | Peter    | Finance   | 1500   | 2000            |
| 6      | Tim      | HR        | 2000   | 0               |
| 7      | Johnson  | HR        | 1000   | 0               |
| 8      | Larry    | HR        | 1500   | 2000            |

### first_value() / last_value() / nth_value()

`first_value()` 获取窗口中第一个 column 中的值，`last_value()` 获取窗口中最后一个 column 中的值。

比如，获取每个部门最高薪资的员工名：

```sql
select 
	emp_id, 
	emp_name, 
	dept_name, 
	salary, 
	first_value(emp_name) over(partition by dept_name order by salary desc) as highest_salary_empt_name_in_dept
from employee_table;
```

SQL 执行结果类似于：

| emp_id | emp_name | dept_name | salary | highest_salary_empt_name_in_dept |
| ------ | -------- | --------- | ------ | -------------------------------- |
| 1      | James    | Finance   | 4000   | John                             |
| 2      | John     | Finance   | 5000   | John                             |
| 3      | Sally    | HR        | 2000   | Mary                             |
| 4      | Mary     | HR        | 3000   | Mary                             |
| 5      | Peter    | HR        | 1500   | Mary                             |

可以知道，按照部门分组后，再根据薪资降序，此时 Finance 最高的薪资是 5000，对应的员工名称是 John。

**但是， <span id='frame_example'>`last_value()` 却有点不一样</span>：**

```sql
select 
	emp_id, 
	emp_name, 
	dept_name, 
	salary, 
	last_value(emp_name) over(partition by dept_name order by salary desc) as lowest_salary_empt_name_in_dept
from employee_table;
```

SQL 执行结果类似于：

| emp_id | emp_name | dept_name | salary | lowest_salary_empt_name_in_dept |
| ------ | -------- | --------- | ------ | ------------------------------- |
| 1      | James    | Finance   | 4000   | James                           |
| 2      | John     | Finance   | 5000   | James                           |
| 3      | Sally    | HR        | 2000   | Sally                           |
| 4      | Mary     | HR        | 3000   | Sally                           |
| 5      | Peter    | HR        | 1500   | Peter                           |

此时，HR 部门最低工资的应该是 Peter，但是直到最后 1 行，才计算出 Peter 是工资最低的。这其实就是因为在使用了 `order by` 后，会有一个默认的 [Frame Clause](#Frame_Clause) —— `range between unbounded preceding and current row` 。

---

`nth_value()` 可以类比 `first_value()` 和 `last_value()`，是用于获取当前 window (frame) 中某个顺序的值。

比如，获取 每个产品分类中，第 5 贵的产品的名称：

```sql
select
	nth_value(product_name, 5) over(
    partition by product_category
    order by price desc
    range between unbounded preceding and unbounded following
  )
from product;
```

此时，如果没有第 5 贵的产品，就会放入 `null`。

如果没有加上 `range between unbounded preceding and unbounded following`，那么每个分组的前 4 行都会是 `null`。这是因为使用了默认的 `range between unbounded preceding and current row` 。

### ntile()

> tile 是瓷砖、瓦片的意思。

`ntile(数值)` 可以将 window 中的内容，尽可能地平分为多个部分。

假设 `select * from phone` 的结果是：

| brand   | product      | price |
| ------- | ------------ | ----- |
| Apple   | iPhone Plus  | 5999  |
| Samsung | Galaxy S     | 5899  |
| Xiaomi  | Mix          | 3899  |
| OPPO    | Find         | 3899  |
| OPPO    | Fold         | 8999  |
| Apple   | iPhone Pro   | 7999  |
| Samsung | Galaxy Ultra | 7899  |

当 SQL 如下时：

```sql
select
	brand,
	product,
	price,
	ntile(3) over() as part_no
from phone;
```

其执行的结果如下，也就是按照正常顺序，将整个 window 分为 3 个部分并加上编号：

| brand   | product      | price | part_no |
| ------- | ------------ | ----- | ------- |
| Apple   | iPhone Plus  | 5999  | 1       |
| Samsung | Galaxy S     | 5899  | 1       |
| Xiaomi  | Mix          | 3899  | 1       |
| OPPO    | Find         | 3899  | 2       |
| OPPO    | Fold         | 8999  | 2       |
| Apple   | iPhone Pro   | 7999  | 3       |
| Samsung | Galaxy Ultra | 7899  | 3       |

如果搭配 `partition by` ，当 SQL 如下时：

```sql
select
	brand,
	product,
	price,
	ntile(2) over(partition by brand) as part_no
from phone;
```

其执行的结果如下，即，分组后，将每个分组内部，分为 2 个部分并加上编号：

| brand   | product      | price | part_no |
| ------- | ------------ | ----- | ------- |
| Apple   | iPhone Plus  | 5999  | 1       |
| Apple   | iPhone Pro   | 7999  | 2       |
| OPPO    | Find         | 3899  | 1       |
| OPPO    | Fold         | 8999  | 2       |
| Samsung | Galaxy S     | 5899  | 1       |
| Samsung | Galaxy Ultra | 7899  | 2       |
| Xiaomi  | Mix          | 3899  | 1       |

同理，如果 SQL 只加上了 `order by` ：

```sql
select
	brand,
	product,
	price,
	ntile(3) over(order by price desc) as part_no
from phone;
```

其执行的结果会先根据 price 倒序排序，再将排序后的结果，分为 3 组，最后加上分组的编号：

| brand   | product      | price | part_no |
| ------- | ------------ | ----- | ------- |
| OPPO    | Fold         | 8999  | 1       |
| Apple   | iPhone Pro   | 7999  | 1       |
| Samsung | Galaxy Ultra | 7899  | 1       |
| Apple   | iPhone Plus  | 5999  | 2       |
| Samsung | Galaxy S     | 5899  | 2       |
| OPPO    | Find         | 3899  | 3       |
| Xiaomi  | Mix          | 3899  | 3       |

### percent_rank() / cume_dist()

`percent_rank()` ：

- 公式 = (当前行的排序的数值 - 1) / (总的行数 - 1)
- 含义：求当前数值的百分位

`cume_dist()`：

- 公式 = (当前行的排序的数值) / (总的行数)
- 含义：Cumulative Distribution Function，累积分布函数、分布函数

假设 `select * from product_table` 的结果为：

| product   | price |
| --------- | ----- |
| Computer  | 6000  |
| Phone     | 3000  |
| Earphones | 1000  |
| Kindle    | 800   |
| Glasses   | 1500  |
| Keyboard  | 500   |
| iPad      | 4000  |

当 SQL 如下时：

```sql
select
	product,
	price,
	percent_rank() over (order by price desc) as per_rank,
	cume_dist() over  (order by price desc) as cu_dist
from product_table;
```

其结果为：

| product   | price | per_rank | cu_dist |
| --------- | ----- | ---- | ---- |
| Computer | 6000 |	0 |	0.14285714285714285 |
| iPad	| 4000 | 0.16666666666666666 |	0.2857142857142857 |
| Phone |	3000 | 0.3333333333333333 | 0.42857142857142855 |
| Glasses |	1500 | 0.5 | 0.5714285714285714 |
| Earphones | 1000 | 0.6666666666666666 | 0.7142857142857143 |
| Kindle | 800 | 0.8333333333333334 | 0.8571428571428571 |
| Keyboard | 500 | 1 | 1 |

## Frame Clause

### <span id='Frame_Clause'>Frame Clause</span> 入门

**A frame is a subset of a partition. Frame 是窗口内部的”窗口“。**

对于使用了 `over(order by)` 的 window function 而言，其**默认的 frame clause 是 `range between unbounded preceding and current row`**。即，如下所示：

```sql
【function()】 over(【 order by ...】 range between unbounded preceding and current row)
```

>等价于 `【function()】 over(【 order by ...】 range unbounded preceding)`
>
>也就是说，如果只有 ` 【range / rows】【unbounded / 数值】 preceding` 的话，就等价于 ` 【range / rows】 between 【unbounded / 数值】 preceding and current row`。即，可以省略 between ... and current row。

以 [上文中的 SQL](#frame_example) 为例，其等价于：

```sql
select 
	emp_id, 
	emp_name, 
	dept_name, 
	salary, 
	last_value(emp_name) over(
    partition by dept_name 
    order by salary desc
    range between unbounded preceding and current row
  ) as lowest_salary_empt_name_in_dept
from employee_table;
```

也就是说，这个 SQL 默认是计算 当前部门的分组里面，第 1 行，到当前行的这个 frame 里面的最低工资。

如果要计算 当前部门的分组里面，所有行的最低工资，需要改为：

```sql
select 
	emp_id, 
	emp_name, 
	dept_name, 
	salary, 
	last_value(emp_name) over(
    partition by dept_name 
    order by salary desc
    range between unbounded preceding and unbounded following
  ) as lowest_salary_empt_name_in_dept
from employee_table;
```

也就是将 frame clause 改为 `range between unbounded preceding and unbounded following`。

Frame clause 除了上面的 `range` 参数，还有 `rows` 参数。

### Frame Clause: Syntax

**Frame clause 定义**：

- `rows` or `range` argument in `over()` clause can **limit the rows within a partition by specifiying the starting and end points of a window frame**.
- This way we can **create fixed size windows which can be used to computing moving averages or running totals with a limited number of rows**.
- If we don't specify the argument, **the default value is from the start of the window frame to the current row**. 也就是 `range between unbounded preceding and current row`

**Frame clause 中，`rows` 和 `range` 的不同**：

- `row` specifies **a fixed number of rows** that precede or follow the current row
- `range` specifies **the range of values** with repect to the value of the current row

假设 `select * from phone` 的结果是：

| brand   | product  | price |
| ------- | -------- | ----- |
| Apple   | iPhone   | 5999  |
| Samsung | Galaxy S | 5899  |
| Xiaomi  | Mix      | 3899  |
| OPPO    | Find     | 3899  |
| OPPO    | Fold     | 8999  |

**当使用 `rows` 参数的时候，frame 指向的 `current row` 就是 literally “当前行”（the exact current row）**：

```sql
select 
	brand, 
	product, 
	price,
	last_value(product) over(
  	order by price desc
  	rows between unbounded preceding and current row
	) as least_exp_product
from phone;
```

执行的结果类似于：

| brand   | product  | price | least_exp_product |
| ------- | -------- | ----- | ----------------- |
| OPPO    | Fold     | 8999  | Fold              |
| Apple   | iPhone   | 5999  | iPhone            |
| Samsung | Galaxy S | 5899  | Galaxy S          |
| Xiaomi  | Mix      | 3899  | Mix               |
| OPPO    | Find     | 3899  | Find              |

如结果说是，当 price 相同时，走到该行，就获取到该行为止的最后一个 price。

> 也就是说， **`rows` 参数表示：按照 `order by` 的顺序走下去，走到哪里，哪里就是当前行**。

---

**当使用 `range` 参数的时候，frame 指向的 `current row`  是<u>值相等的多个当前行</u>**：

```sql
select 
	brand, 
	product, 
	price,
	last_value(product) over(
  	order by price desc
  	range between unbounded preceding and current row
	) as least_exp_product
from phone;
```

执行的结果类似于：

| brand   | product  | price | least_exp_product |
| ------- | -------- | ----- | ----------------- |
| OPPO    | Fold     | 8999  | Fold              |
| Apple   | iPhone   | 5999  | iPhone            |
| Samsung | Galaxy S | 5899  | Galaxy S          |
| Xiaomi  | Mix      | 3899  | Find              |
| OPPO    | Find     | 3899  | Find              |

---

**Frame Clause 除了 `unbounded preceding` 、`unbounded following` 和 `current row` 之外，还可以使用 `1 preceding` 和 `2 following` 来分别表示“前 1 个”和“后 2 个”** （其他的，以此类推）。

这样就可以实现 **滑动窗口**了。

### Window Function

Frame clause 还有一种替代写法，也就是使用 Window Function。

普通的写法：

```sql
select 
	brand, 
	product, 
	price,
	first_value(product) over(
    partition by product_catagory
  	order by price desc
  	range between unbounded preceding and unbounded following
	) as most_exp_product,
	last_value(product) over(
    partition by product_catagory
  	order by price desc
  	range between unbounded preceding and unbounded following
	) as least_exp_product
from phone
where product_catagory = 'Phone'
order by product_id;
```

而 Window Function 的写法为：

```sql
select 
	brand, 
	product, 
	price,
	-- 这里不是 over(...) 而是 over 加上后面 window 的别名用作窗口函数
	first_value(product) over win as most_exp_product,
	last_value(product) over win as least_exp_product
from phone
where product_catagory = 'Phone'
-- 在 where 后面、order by 前面，可以使用：window 【window的别名】as (【窗口中的内容】)
window win as (
    partition by product_catagory
  	order by price desc
  	range between unbounded preceding and unbounded following
	)
order by product_id;
```

也就是将 `over()` 中的 `()` 里面的内容抽取出来放到后面。

# 常用函数

## timestampdiff

`timestampdiff(时间单位, 开始时间, 结束时间)`

> 注意，第 2 个参数是开始时间，第 3 个参数是结束时间。计算公式为：结束时间 - 开始时间

比如：`timestampdiff(minute, start_time, end_time)` 。假设数据只有一行，`start_time` 为  `2021-09-01 06:00:00`，`end_time` 为 `2021-09-01 06:05:00` ，那么求出的结果为 5，表示过了 5 分钟。

## ifnull / nvl

> MySQL 中是 `ifnull` ，Oracle 中是 `nvl`。
> 下文以 `ifnull` 为例。

用法：ifnull(不为null时的值, 为null时的值)

例子：

```sql
SELECT IFNULL(1, 0); -- returns 1

SELECT IFNULL('', 1); -- returns ''

select ifnull(this_is_val, 0);
-- 当 this_is_val 不是 null 的时候，返回 this_is_val
-- 当 this_is_val 是 null 的时候，返回 0
```

# 实用语法

## exists 语法

> 内容整理自 ChatGPT
>
> `not exists` 的用法可参考 `exists`，这里就不赘述了。

`EXISTS` 语法用于检查一个子查询中是否存在任何行。

具体语法格式如下：

```sql
SELECT column_name(s)
FROM table_name
WHERE EXISTS
(SELECT column_name FROM table_name WHERE condition);
```

其中，column_name 是要选取的列名，table_name 是要查询的表名，condition 是任何适用的限制条件。

下面的例子，假设有两张表：orders 和 customers，orders 表中存储着订单信息，包括订单 ID（order_id）、客户 ID（customer_id）、订单日期（order_date）等等，customers 表中存储着客户的信息，包括客户 ID（customer_id）、客户姓名（customer_name）、联系电话（phone_number）等等。

现在，我们想要查询所有已经下了订单的客户的姓名和联系电话，可以使用 EXISTS 语法实现。具体查询语句如下：

```sql
SELECT customer_name, phone_number
FROM customers
WHERE EXISTS
(SELECT NULL FROM orders WHERE customers.customer_id = orders.customer_id);
```

在这个例子中，我们使用了子查询来检查 customers 表和 orders 表中是否存在相同的 customer_id，如果存在，则返回该客户的姓名和联系电话。

其中，子查询返回的值是 NULL，这是因为我们只关心该子查询是否返回了任何结果，而不需要具体的值。

> 个人补充：也就是说，这里直接 `select null` 是完全没问题的
>
> 个人理解：只要子查询里面的 select 语句找到任意一条符合的结果，就会立刻返回 true 并停止全表扫描

---

如果您想查询客户是否存在订单，可以使用 hasOrder 方法：

```java
@Select("SELECT EXISTS(SELECT 1 FROM orders WHERE customer_id=#{customerId})")
boolean hasOrder(Integer customerId);
```

在上述代码中，hasOrder 方法使用 EXISTS 子查询检查指定客户是否有订单，如果存在订单则返回 true，否则返回 false。

查询结果将会映射为 Java 中的布尔值类型，即 true 或 false。

> 个人补充：
>
> 查询在 bookmark 表中，是否有 id 为 2 的数据，可以使用：`select exists(select null from bookmark where id = 2);`
>
> 如果返回 1，就表示存在；返回 0 就表示不存在。

## any / all 语法

`any` 和 `all` 的区别：

1. `any` 表示满足其中一个

2. `all` 要全部都满足

使用上没什么区别。下文就以 `any` 为例即可。

> 下文整理自 ChatGPT

`ANY` 是 SQL 中的一个操作符，用于比较子查询的结果与给定值的关系。它用于过滤、匹配和比较表格中的数据。

`ANY` 可以用于以下情况：

1. 与比较操作符一起用于比较子查询的结果和一个值：

   例如：查找表格中销售量大于任意单个业绩表中相应的值的产品。以下是一个示例查询：

```sql
SELECT *
FROM products
WHERE units_sold > ANY (
   SELECT sales_units
   FROM quarterly_sales
   WHERE quarter = '2021-Q1'
);
```

这将检查表格中的每个行，如果其销售量高于 2021 年第一季度业绩表格中的任何一个单元的值，则返回行。

> 个人补充：这是最基本的用法，MySQL 和 Oracle 都支持

2. 类似 `IN` 运算符，用于查询满足条件的数据。

   例如：查询表格中所有订单的客户名，如果客户名在指定列表中，且订单数量大于 20。以下是一个示例查询：

```sql
SELECT customer_name
FROM orders
WHERE order_quantity > 20
AND customer_name = ANY (
   'Alice',
   'Bob',
   'Charlie'
);
```

这将返回具有订单数量大于 20 的所有客户，且客户名称是 Alice、Bob 或 Charlie 的订单。

> 个人补充：在 MySQL 中无法使用。

需要注意的是，`ANY` 是比较操作符，对于不同的 SQL 实现，其语法和具体语义可能会有所不同。因此，应该根据具体情况学习如何正确使用 `ANY`。

---

个人总结：

1. `any` 语法可以搭配（`>` , `!=` 和 `=` 等）比较运算符，来比较 `select` 子查询中的结果，并返回符合比较结果的值
2. 也可以不用子查询，而是直接需要比较的值列出来。这种情况下，MySQL 无法使用。

## ON DUPLICATE KEY UPDATE：不能插入就更新

如下面的例子（使用了 MyBatis），当 `biz_tag, user_id` 为 generate_id_tbl 表的 primary key（这里使用主键或唯一索引约束都是一样的道理）时：

```sql
insert into generate_id_tbl(biz_tag, user_id, max_id, step)
values (#{biz_tag}, #{userId}, 3000, 1000)
ON DUPLICATE KEY UPDATE max_id = max_id + step
```

如果 `insert` 时触发了约束，也就是出现 `DUPLICATE KEY` ，就只会执行 `update` 操作，不会 `insert` 。

如果 `insert` 时没有触发 `DUPLICATE KEY` ，就执行 insert 的操作，而不执行 `update` 操作。

# 实用技巧

## 在 sum() 中，搭配 case when、使用负数

slow_rank 表示按照 从慢到快 排序后的 column，fast_rank 表示按照 从快到慢 排序后的 column，used_time 表示 经过的时间（比如，5 分钟）。这里要求的 diff_between_2nd_slow_and_2nd_fast 表示第 2 慢的时间 - 第 2 快的时间：

```sql
sum(
	case
    	-- 这里是通过 sum 来求【第2慢的时间-第2快的时间】
    	-- 当时间为第 2 慢 时，sum() 的时候是正数的时间
    	-- slow_rank = 2 表示【第2慢】，used_time 表示 【第2慢的时间】
		when slow_rank = 2 then used_time
    	-- 当时间为第 2 快 时，sum() 的时候是负数的时间
    	-- fast_rank = 2 表示【第2快】，-used_time 表示 【-第2快的时间】
    	-- sum() 的时候，就是 第 2 慢 - 第 2 快
		when fast_rank = 2 then - used_time
    	-- 其他的值，sum() 的时候设置为 0
		else 0
	end
) as diff_between_2nd_slow_and_2nd_fast
```

