---
title: 笔记：MySQL 基础
date: 2025-04-06
categories:
  - 数据管理
  - 关系型数据库
  - MySQL
  - MySQL 基础
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. 导图：[Map：MySQL](Map：MySQL.xmind)

---
![](image-20250615085438498.png)

### 2. 数据库核心

数据库管理系统、数据库、表

---


### 3. MySQL 最佳实践

---


### 4. MySQL 语法

#### 4.1. 语法要求

1. SQL 语句可以单行或多行，但要以分号（`;`）结尾
2. MySQL 默认情况下，每条 SQL 语句的最大长度为 4MB

---


#### 4.2. 注释写法

```
# 单行注释

-- 单行注释

/*
多行注释
*/
```

---


#### 4.3. DDL（库、表、列）

##### 4.3.1. 库 操作

==1.创建数据库==
```
create database [if not exists] <repository-name> [default charset 字符集] [collate 排序规则];
```

> [!NOTE] 注意事项
> 1. 若不指定字符集，默认使用 utf8mb4；
> 2. 若不指定排序规则，默认使用 utf8mb4_0900_ai_ci


==2.删除数据库==
```
drop database [if exists] <repository-name>;
```


==3.更新数据库==
```
# 1. 修改数据库名
# 1.1. 导出旧数据库（将旧数据库导出到一个 SQL 文件中)
mysqldump -p <old-repository-name> > <old-repository-name> .sql
	
# 1.2. 创建新的数据库
CREATE DATABASE <new-repository-name>;
	
# 1.3. 将旧库文件导入新库
mysql -p <new-repository-name> < <new-repository-name>.sql

# 1.4. 删除旧数据库
DROP DATABASE <old-repository-name>;
```


==4.查询数据库==
```
# 1. 查看所有数据库
show databases;


# 2. 查看当前使用的数据库
select databses();
```


==5.使用数据库==
```
use <repository-name>;
```

---


##### 4.3.2. 表 操作

==1.创建表==
```
CREATE TABLE [IF NOT EXISTS] <table-name> (
    <id-column> <数据类型> AUTO_INCREMENT PRIMARY KEY [COMMENT 'explanation for this column'],
    <column-name> <数据类型> NOT NULL [DEFAULT <默认值>] [COMMENT 'explanation for this column'],
    <column-name> <数据类型> NOT NULL [DEFAULT <默认值>] [COMMENT 'explanation for this column']
) [ENGINE=<engine-type>] [DEFAULT CHARSET=<字符集>] [COMMENT='explanation for this table'];
```


==2.删除表==
```
drop table [if exists] <table-name>;
```


==3.更新表==
```
# 1. 修改表名
alter table <old-table-name> rename to <new-table-name>; 
```


==4.查询表==
```
# 1. 查看当前使用的数据库所有表
show tables;


# 2. 查看表结构
desc <table-name>


# 3. 查看建表语句
show create table <table-name>;
```

---


##### 4.3.3. 列 操作

==1.创建列==
```
alter table <table-name> add <column-name> <数据类型> [comment '列的注释']
```


==2.删除列==
```
alter table <table-name> drop column <column-name>
```


==3.修改列==
```
# 1. 修改列名
alter table <table-name> rename column <old-column-name> to <new-column-name>
```

---


#### 4.4. DML（数据增删改）

##### 4.4.1. 插入数据

###### 4.4.1.1. 数据插入原则
1. ==主键顺序插入==：
	1. 插入数据时，最好安装主键的递增顺序进行插入，因为顺序插入的性能通常高于乱序插入。这可以减少索引的重组和提高插入的效率
2. ==开启事务==：
	1. 大量数据插入时，建议使用事务以确保数据的一致性和完整性。这样可以避免由于中途失败而导致的数据不一致问题，确保操作的原子性。
3. ==以行为计==：
	1. 插入数据都是以行为单位，即插入一行或者多行，不能插入某一行的某一条数据

---


###### 4.4.1.2. 普通插入数据

```
# 1. 模版
insert into <table-name> [<column1,column2,...>] values <value1,value2,......>


# 2. 指定字段添加数据
insert into students (name, age) values ('Alice', 20);


# 3. 全部字段添加数据
insert into students values (NULL, 'Bob', 22);


# 4. 批量添加数据
insert into students values 
(NULL, 'Alice', 20),
(NULL, 'Bob', 22),
(NULL, 'Charlie', 21),
(NULL, 'David', 23),
(NULL, 'Eva', 19);


# 5. 主键回显
# 5.1. 执行 inset 语句
INSERT xxxxxxx

# 5.2. 获取上一条插入记录在表中的主键值
SELECT LAST_INSERT_ID();
```

> [!NOTE] 注意事项：主键回显
> 1. 执行插入语句后，可以通过执行 `SELECT LAST_INSERT_ID()` 获取上一条插入记录的主键值。这样做的主要原因是，插入记录后，我们通常需要立即对该记录进行后续操作。如果再次使用 `SELECT` 查询来获取其 ID，这会增加额外的操作开销。通过主键回显，我们可以直接获得该记录的 ID，提高效率。

---


###### 4.4.1.3. 千万级插入数据

---


##### 4.4.2. 删除数据

删除数据同样是以行为单位，即删除一行或者多行，不能删除某一行的某一条数据
```
# 1. 删除部分数据
delete from <table-name> [where condition];


# 2. 删除整个表的数据（高效）
truncate table <table-name>;


# 3. 删除分区来删除数据（高效）
alter table <table-name> drop partition <partition-name>
```

> [!NOTE] 注意事项：
> 1. 如果使用 `delete from` 不加条件，它会删除整个表的数据，但这种方法效率较低，不推荐使用。如果必须采用此方法，建议分批删除（例如：`delete from table where id between 1 and 2000`），并**开启事务**以确保数据一致性和可靠性。
> 2. 方法二和方法三不需要开启事务，因为他们都很高效

---


##### 4.4.3. 更新数据
```
update <table-name> set <column1=value1>,<column2=value2>,.... [where condition];
```

> [!NOTE] 注意事项
> 1. 如果需要批量更新数据，同样应该**开启事务**

---


#### 4.5. DQL（数据查询）

##### 4.5.1. 数据查询模版

```
select [all | distinct] 
{ * | <column-name> [as 列别名] | <表达式 [as 表达式别名]> }       # 选择列表，表达式支持聚合函数
from <table-name> [as 表别名]                                    # 普通查询
[<join-type> join <other-table> [as 表别名] on <连接条件>]        # 多表联查
[where <行级过滤条件>]                                            # 过滤条件不可使用聚合函数
[group by <column-name> [having <组级过滤条件>]]                  # 分组查询，过滤条件可使用聚合函数
[order by <column-name> [asc | desc]]                            # 排序查询
[limit <偏移量>,<行数>]                                           # 分页查询
```

> [!NOTE] 注意事项
> 1. 一定要有这样的思想，除了普通查询，其他的都是 JB 条件查询懂吗，包括子查询，例如找出每个部分工资最高的员工，条件就是：员工是要每个部门工资最高的有两个条件，





> [!NOTE] 注意事项：
> 2. <font color="#00b0f0">子查询</font>：
> 	- 子查询的结果可以看作是一个**虚拟表**，**类似于视图**，之后可以在这个虚拟表上进行进一步的查询操作
>     - `virtual-table-alias` 就是我们为这个虚拟表起的别名，方便在后续引用这个表
> 	- 子查询内部还可以包含其他子查询，实现多层嵌套，但需注意避免无限嵌套
> 3. <font color="#00b0f0">distinct</font>：
> 	- 用于去除重复记录
> 4. <font color="#00b0f0">SQL 执行顺序（☆☆☆）</font>：
> 	- FROM -> JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT / OFFSET
> 5. <font color="#00b0f0">condition 怎么写</font>:
> 	- 
> 6. <font color="#00b0f0">having-condition 怎么写</font>:
> 	- 
可以有多个字段，用逗号隔开
> [!NOTE] 注意：`WHERE` 与 `HAVING` 的区别
> 1. 执行时机不同
> 2.  `WHERE` ：在分组之前进行条件过滤，不满足条件的记录不会参与分组。
> 3.  `HAVING`：在分组之后对结果进行过滤。
> 4. 判断条件不同
> 5. 切操作需遵循 SQL 执行顺序。在为表起别名之前，我们可以直接使用表名；但一旦为表指定了别名，之后必须使用别名，不能再使用原表名。
> 6. 列明类？
> 7. 
> 8.  `WHERE` 不能对聚合函数进行判断，而 `HAVING` 可以

---


##### 4.5.2. 基本查询

```
select distinct id, order_date as orderDate from my_table where id >20; 

select * from my_table
```

---


##### 4.5.3. 分组查询

###### 4.5.3.1. 分组查询概述

关于分组查询，首先我们要理解分组的概念：可以这样简单理解，当我们在一张表中，发现**某一列的值出现了多次**，我们就可以按照这一列的值把数据分成一组一组的，这就是“分组”的基本概念。

举个例子：假设你是一家公司的老板，你有很多员工，每个员工都有所属的部门（比如：技术部、市场部、财务部……）。现在你想统计每个部门一个月要发多少工资 —— 这时候，你就可以按照“部门”这个字段来进行分组。也就是说：把属于同一个部门的员工聚在一起（分组），然后对每组员工的工资进行汇总（比如求和）。
```
# 1. 工资超过 100 元的员工所在部门的 部门总工资
select department, count(staff_id) as staff_count, sum(amount) as total_amounts
from employees
where amount > 100
group by department
having sum(amount) > 350;


# 2. 统计各个工作地址上班的男性员工和女性员工的数量（根据工作地址、性别分组）、
select workaddress,gender,count(*) 'amount' from emp group by gender,workaddress;


# 3. 查询年龄小于45的员工，根据不同工作地址，获取员工数量大于3的工作地址（根据工作地址分组）
select workaddress,count(*) address_count from emp where age < 45 group by workaddress having address_count > 3;
```

> [!NOTE] 注意事项
> 1. 分组查询 `99%` 的情况下，要与函数结合进行查询

---


###### 4.5.3.2. 实际案例

假设我们有一个员工表 `employees`，包含 `staff_id`（员工ID）、`department`（部门）和 `amount`（工资）等字段，现在我们要统计每个部门中，需要下发工资总额大于 400 的部门。
```
select department, count(staff_id) as staff_count, sum(amount) as total_amounts
from employees
group by department
having sum(amount) > 350;
```
![](source/_posts/笔记：MySQL%20基础/image-20250410141156407.png)



==1.SQL 执行：FROM -> JOIN -> WHERE==
![](source/_posts/笔记：MySQL%20基础/image-20250410141156407.png)


==2.SQL 执行：-> GROPY BY==
可以**理解为**，相同组的数据被归类到一起，形成了很多表（当然这是不可能的）例如：

1. **归类1**
![](source/_posts/笔记：MySQL%20基础/image-20250410145608083.png)

2. **归类2**
![](source/_posts/笔记：MySQL%20基础/image-20250410145637852.png)

3. **归类3**
![](source/_posts/笔记：MySQL%20基础/image-20250410145648701.png)

4. **归类4**
![](source/_posts/笔记：MySQL%20基础/image-20250410145658659.png)

5. **归类5**
![](source/_posts/笔记：MySQL%20基础/image-20250410145805811.png)


==3.SQL 执行：-> HAVING==
HAVING 对前面分组后的数据进行了筛选。

1. **筛选1**
![](source/_posts/笔记：MySQL%20基础/image-20250410145817283.png)

2. **筛选2**
![](source/_posts/笔记：MySQL%20基础/image-20250410145951287.png)


==4.SQL 执行：-> SELECT -> DISTINCT -> ORDER BY -> LIMIT / OFFSET==
`SELECT` 可以**理解为**对前面筛选出来的多个表分别执行查询操作，然后将结果聚合成一个表。

1. **查询1**
![](source/_posts/笔记：MySQL%20基础/image-20250410150044456.png)

2. **查询2**
![](source/_posts/笔记：MySQL%20基础/image-20250410150052131.png)

3. **聚合**
![](source/_posts/笔记：MySQL%20基础/image-20250410150102998.png)

---


##### 4.5.4. 排序查询

```
# 1. 根据年龄，对员工进行升序排序
select * from emp order by age;


# 2. 根据入职时间，对员工进行降序排序
select * from emp order by entrydate desc;


# 3. 根据年龄对员工进行升序排序，若年龄相同，再按照入职时间进行降序排序
select * from emp order by age asc,entrydate desc;
```

> [!NOTE] 注意事项
> 1. <font color="#00b0f0">排序方式</font>：
> 	- <font color="#7030a0">ASC（默认）</font>：升序
> 	- <font color="#7030a0">DESC</font>：降序
> 1. 若有多个字段，先按第一个字段排序；若第一个字段中存在相同值，则在这些相同值的数据行中，按第二个字段继续排序。

---


##### 4.5.5. 分页查询

表的记录可能会有几百万到上亿条记录，显示所有数据是不现实和低效率的，所以我们通常会在网站中进行分页，每页的数据就是根据分页查询做到的

计算方法：设起始行数为 n ，行数为 m ，从 n + 1 开始显示数据的，共显示 m 行数据

```
# 1. 若每页 6 条记录，查询第一页数据
select * from emp limit 0,6;
select * from emp limit 6;

# 2. 若每页 6 条记录，查询第二页数据
select * from emp limit 6,12;

# 3. 若每页 6 条记录，查询第三页数据
select * from emp limit 12,6;
```

---


##### 4.5.6. 多表联查

###### 4.5.6.1. 多表联查和外键约束的关系

多表联查和外键约束本身并没有直接关系。外键约束主要用于保证数据的完整性，而不会直接影响联查的执行。

然而，进行多表联查时，通常是基于外键字段进行匹配查询的。例如，表 A 的 `dept_id` 外键与表 B 的 `dept_id` 进行关联，我们一般通过 `a.dept_id = b.dept_id` 来匹配数据。

---


###### 4.5.6.2. 笛卡尔积问题

执行多表查询时，若直接使用 `select * from emp,dept;` 执行结果有笛卡尔积问题：
![](source/_posts/笔记：MySQL%20基础/image-20250410161532467.png)


在 SQL 语句中，如何去除无效的笛卡尔积？给多表查询加上连接查询的条件即可 `select * from emp,dept where emp.dept_id = dept.id;` 执行结果如下：
![](source/_posts/笔记：MySQL%20基础/image-20250410161552865.png)

---


###### 4.5.6.3. 数据准备

假设我们有 `employees`（员工表）和 `departments`（部门表）：

==1.employess 表==
![](source/_posts/笔记：MySQL%20基础/image-20250410165218847.png)


==2.departments 表==
![](source/_posts/笔记：MySQL%20基础/image-20250410162851472.png)

---


###### 4.5.6.4. 内连接

内连接是最常用的连接方式。当连接条件（如 `e.department_id = d.department_id`）满足时，它会返回两个表中所有符合条件的行及其所有列（满足才返回，不满足不返回）。之后，我们可以对结果应用 `SELECT` 和 `HAVING` 进行进一步筛选，最终得到查询结果。
```
select 
    e.first_name, 
    e.last_name, 
    e.salary, 
    d.department_name,
    d.location
from 
    employees as e
inner join 
    departments as d on e.department_id = d.department_id;
```
![](source/_posts/笔记：MySQL%20基础/image-20250410165304264.png)
> [!NOTE] 注意事项
> 1. 根据 SQL 执行顺序，先执行 `FROM`，再执行 `JOIN`，因此在 `FROM` 和 `INNER JOIN` 中为表指定的别名，可以在整个查询中使用。
> 2. 切操作需遵循 SQL 执行顺序。在为表起别名之前，我们可以直接使用表名；但一旦为表指定了别名，之后必须使用别名，不能再使用原表名。

---


###### 4.5.6.5. 左连接

左连接会返回左表的所有行和所有列，无论连接条件是否满足。当连接条件（如 `e.department_id = d.department_id`）满足时，右表会返回匹配的行及其所有列；如果连接条件不满足（如 `e.department_id != d.department_id`），右表的列则会返回 `NULL`。之后，我们可以对结果应用 `SELECT` 和 `HAVING` 进行进一步筛选，最终得到查询结果。
```
select 
    e.first_name, 
    e.last_name, 
    e.salary, 
    d.department_name
from 
    employees e
left join 
    departments d on e.department_id = d.department_id;
```
![](source/_posts/笔记：MySQL%20基础/image-20250410165415783.png)

---



###### 4.5.6.6. 右连接

和左连接类似，只不过是右表提供所有数据
``` 
select 
    e.first_name, 
    e.last_name, 
    e.salary, 
    d.department_name
from 
    employees e
right join 
    departments d on e.department_id = d.department_id;
```
![](source/_posts/笔记：MySQL%20基础/image-20250410165544009.png)

---


###### 4.5.6.7. 自连接

自连接是将同一表与自身进行连接，常用于需要比较同一表中不同记录的情况。
``` 
SELECT a.column1, b.column1
FROM table_a a
INNER JOIN table_a b ON a.common_column = b.common_column
WHERE a.id != b.id;  -- 防止自身匹配
```

---


###### 一对一查询

我们上面讲的内连接、左连接、右连接都很适合进行一对一查询。以 `Employee` 表和 `WorkCard` 表为例，`Employee` 表中的每个员工都有一个唯一的 `EmpID`，而 `WorkCard` 表中的每个工作证都对应一个唯一的员工。
```
# 1. Employee 表（主表）
CREATE TABLE Employee (
    EmpID INT PRIMARY KEY,
    Name VARCHAR(100)
);


# 2. WorkCard 表（从表）
CREATE TABLE WorkCard (
    CardID INT PRIMARY KEY,
    EmpID INT, // 从表的外键字段 -> 主表的字段（一般命名相同）
    IssueDate DATE,
    FOREIGN KEY (EmpID) REFERENCES Employee(EmpID)
);


# 3. 一对一查询（以内连接为例）
SELECT 
  e.EmpID,
  e.Name,
  w.CardID,
  w.IssueDate
FROM 
  WorkCard w
INNER JOIN 
  Employee e // FROM 从 JOIN 主
ON 
  e.EmpID = w.EmpID;
```
![](source/_posts/笔记：MySQL%20基础/image-20250521211043750.png)

![](source/_posts/笔记：MySQL%20基础/image-20250521211108505.png)

----


###### 一对多查询

我们上面讲的内连接、左连接、右连接同样也都很适合进行一对多查询。假设有 `Department`（部门表）和 `Employee`（员工表）。一个部门有多个员工，每个员工属于一个部门。
```
# 1. Department 表（主表）
CREATE TABLE Department (
    DeptID INT PRIMARY KEY,
    DeptName VARCHAR(100)
);


# 2. Employee 表（从表）
CREATE TABLE Employee (
    EmpID INT PRIMARY KEY,
    Name VARCHAR(100),
    DeptID INT, // 从表的外键字段 -> 主表的字段（一般命名相同）
    FOREIGN KEY (DeptID) REFERENCES Department(DeptID)
);


# 3. 一对多查询（以内连接为例）
SELECT 
  e.EmpID,
  e.Name,
  d.DeptName
FROM 
  Employee e
INNER JOIN 
  Department d
ON 
  e.DeptID = d.DeptID;
```
![](source/_posts/笔记：MySQL%20基础/image-20250521212132677.png)

![](source/_posts/笔记：MySQL%20基础/image-20250521212154361.png)

---


###### 多对多查询

多对多查询的关键在于：查中间表，并对左右两侧的主表各做一次连接。假设有 `Student`（学生表）和 `Course`（课程表），一个学生可以选修多门课程，而每门课程可以有多个学生选修。我们需要一个中间表 `Enrollment` 来记录学生与课程之间的关系。
```
# 1. Student 表（主表）
CREATE TABLE Student (
    StudentID INT PRIMARY KEY,
    Name VARCHAR(100)
);


# 2. Course 表（主表）
CREATE TABLE Course (
    CourseID INT PRIMARY KEY,
    CourseName VARCHAR(100)
);


# 3. Enrollment 表（从表）
CREATE TABLE Enrollment (
    StudentID INT,
    CourseID INT,
    PRIMARY KEY (StudentID, CourseID),
    FOREIGN KEY (StudentID) REFERENCES Student(StudentID),
    FOREIGN KEY (CourseID) REFERENCES Course(CourseID)
);


# 4. 多对多查询
SELECT
  s.Name AS 学生名,
  c.CourseName AS 课程名
FROM
  Enrollment e // 中间表
JOIN
  Student s ON e.StudentID = s.StudentID // 第一次 JOIN：连接学生
JOIN
  Course c ON e.CourseID = c.CourseID // 第二次 JOIN：连接课程
```
![](source/_posts/笔记：MySQL%20基础/image-20250522101521175.png)

![](source/_posts/笔记：MySQL%20基础/image-20250522101539693.png)

![](source/_posts/笔记：MySQL%20基础/image-20250522101559348.png)

> [!NOTE] 注意事项
> 1. 我们平时常见的是两个实体之间的多对多关系，但其实三个甚至更多实体之间也能形成多对多关系，也是一个中间表。查询的时候，只要用中间表，多次连接相关表，就能把信息都查出来。

----


##### 子查询

子查询可以出现在 SQL 的三个位置。现在你只需要先记住最常见的一种用法：写在 `WHERE` 后面，用于筛选数据。

| 放置位置       | 示例                                            | 作用       |
| ---------- | --------------------------------------------- | -------- |
| `WHERE` 后  | `WHERE id IN (SELECT id FROM ...)`            | 筛选数据     |
| `SELECT` 中 | `SELECT (SELECT AVG(score) ...) AS avg_score` | 字段值计算    |
| `FROM` 中   | `FROM (SELECT ...) AS temp`                   | 临时表、组合逻辑 |

当你看到以下几种场景时，通常意味着可能要用到子查询了：
1. ==存在性判断==：
	1. 关键词：
		1. 存在、不存在、是否在，
		2. 简单来说，就是在问 “**有没有？**” 
	2. 例如示例：
		1. 哪些项目没有使用红色零件？
		2. 找出未使用红色零件的项目
	3. 注意事项：
		1. 如果是“有”的情况（如：某些项目使用了红色零件），通常 直连 即可。
		2. 如果是“没有”的情况（如：哪些项目没有使用红色零件），大概率需要用 子查询 来排除。
2. ==极值查询==：
	1. 关键词：
		1. 最大、最小、前几名
3. ==集合比较==：
	1. 关键词：
		1. 全部、至少、包含
4. ==字段间的比较用到全表的动态值==：

---


#### 4.6. DCL（用户、角色管理）

##### 4.6.1. 用户管理

==1.创建用户==
```
create user '<用户名>'@'<主机地址>' identified by '密码'
"""
1. 主机地址：
	1. 主机地址可以为 %，允许该用户从任何主机连接到 MySQL 数据库
	2. 例如：'user'@'%'，表示无论什么 IP，只要你是 user 用户，就都能连接到数据库
"""
```


==2.删除用户==
```
drop user '用户名'@'主机地址';
```


==3.更新用户==
```
# 1. 修改用户密码
alter user '用户名'@'主机地址' identified with mysql_native_password by '新密码';
```


==4.查询用户==
```
# 1. 查询有哪些用户
select * from mysql.user;


# 2. 查询用户的权限
show crants for '用户名'@'主机地址';
```


==5.授予用户权限==
```
# 1. 授予用户权限
grant <权限列表,> on <数据库名>.<表名> to '<用户名>'@'<主机名>' [with grant option];
"""
1. 注意事项：
	1. 一个语句一个 <数据库名>.<表名>，如果你非要同时授权多个表，那就得多写几条语句
	2. 但是我们可以使用 * 表示所有数据库或所有表
2. 常用权限：
3. with grant option：
	1. 用户可以把自己的权限再授权给别的用户

"""


# 2. 刷新权限，使其生效
FLUSH PRIVILEGES;
```

> [!NOTE] 注意事项：常用权限
> 1. <font color="#00b0f0">ALL PRIVILEGES</font>：授予所有权限
> 2. <font color="#00b0f0">ALTER</font>：修改数据库、表的结构
> 3. <font color="#00b0f0">ALTER ROUTINE</font>：修改存储过程或函数
> 4. <font color="#00b0f0">CREATE</font>：创建数据库、表等对象
> 5. <font color="#00b0f0">CREATE ROUTINE</font>：创建存储过程或函数
> 6. <font color="#00b0f0">CREATE TEMPORARY TABLES</font>：创建临时表
> 7. <font color="#00b0f0">CREATE VIEW</font>：创建视图
> 8. <font color="#00b0f0">DELETE</font>：删除数据
> 9. <font color="#00b0f0">DROP</font>：删除数据库、表、视图等对象
> 10. <font color="#00b0f0">EVENT</font>：管理事件调度
> 11. <font color="#00b0f0">EXECUTE</font>：执行存储过程或函数
> 12. <font color="#00b0f0">GRANT OPTION</font>：授予权限
> 13. <font color="#00b0f0">INDEX</font>：创建和删除索引
> 14. <font color="#00b0f0">INSERT</font>：插入数据
> 15. <font color="#00b0f0">LOCK TABLES</font>：锁定表
> 16. <font color="#00b0f0">REFERENCES</font>：管理外键约束
> 17. <font color="#00b0f0">SELECT</font>：查询数据
> 18. <font color="#00b0f0">SHOW VIEW</font>：查看视图的定义
> 19. <font color="#00b0f0">TRIGGER</font>：创建和删除触发器
> 20. <font color="#00b0f0">UPDATE</font>：更新数据
> 21. <font color="#00b0f0">FILE</font>：读取文件
> 22. <font color="#00b0f0">REPLICATION CLIENT</font>：管理复制客户端，允许查看复制状态
> 23. <font color="#00b0f0">REPLICATION SLAVE</font>：允许从服务器连接到主服务器进行复制
> 24. 权限的授予通常基于数据库、表或列的范围。例如，如果你拥有 <font color="#00b0f0">CREATE</font> 权限并且授予到`*.*`，那么你可以创建数据库和表；但如果权限只授予到 `my_repository.*`，你只能创建该库下的表，不能创建数据库；如果是 `my_repository.my_table`，你连表都无法创建。


==6.撤销用户权限==
```
revoke <权限列表,> on <数据库名>.<表名> from '<用户名>'@'<主机名>';
```

---


##### 4.6.2. 角色管理

==1.创建角色==
```
CREATE ROLE '角色名';
```


==2.删除角色==
```
drop role '角色名列表(,)';
```


==3.给角色赋权限==
```
grant 权限列表(,) on 库名.表名 to '角色名列表(,)';
```


==4.把角色分配给用户 / 角色==
```
# 1. 把角色分配给角色
grant '角色名列表(,)' to '角色名列表(,)';


# 2. 把角色分配给用户
grant '角色名列表(,)' to '用户名'@'主机地址'列表(,);
```


==5.将分配给用户的角色撤回==
```
revoke '角色名列表(,)' from '用户名'@'主机地址'列表(,);
```

---


### 5. 事务管理

#### 5.1. 事务的特性
1. ==原子性==
	1. 事务中的所有操作要么全部执行成功，要么全部执行失败回滚。
	2. 对于数据库来说，事务是一个不可分割的最小操作单元。
2. ==一致性==：
	1. 事务执行前后，数据库都处于一致的状态。
	2. 也就是说，事务执行前后，数据库的状态应该满足所有的定义约束、触发器、级联操作等。
3. ==隔离性==：
	1. 事务的执行不会受到其他事务的干扰。
	2. 多个事务并发执行时，一个事务的中间状态对其他事务是不可见的。
4. ==持久性==：
	1. 事务一旦提交，其结果就永久保存到数据库中。
	2. 即使系统发生故障，事务的结果也不会丢失。

---


#### 5.2. 事务提交的方式

一句话：我不管你是**一次操作批量插入**还是**多次操作大量插入**，只要涉及到多个数据，就给我开启手动提交：

1. ==自动提交==：
	1. 在 MySQL 中，默认情况下，每条 SQL 语句都被视为一个独立的事务，并在执行后立即自动提交。
	2. 这意味着每条 SQL 语句执行后，数据库会自动处理提交，无需显式使用 `COMMIT` 命令。
	3. 在这种模式下，操作是独立的，无法将多个操作作为一个整体事务进行管理，需要一条一条的执行。
2. ==手动提交==：
	1. 关闭自动提交，改为手动管理事务，使我们能够自行决定何时提交一组操作。如果其中任一操作失败，事务将回滚至初始状态，确保数据一致性。

> [!NOTE] 注意事项
> 1. 事务只对 DML 语句有影响，原因是：
> 	- DQL 不受事务影响，因为 DQL 不会对数据库进行改变
> 	- DDL 和 DCL 不受事务一下，纯粹是事务管不到他们

----


#### 5.3. 自动提交事务

在 MySQL 中，默认启用了自动提交模式：
```
# 1. 查询是否开启自动提交（1开，0关）
select @@autocommit;


# 2. 开启自动提交
set autocommit = 1
```

---


#### 5.4. 手动提交事务

```
# 1. 查询是否开启自动提交（1开，0关）
select @@autocommit;


# 2. 关闭自动提交
set autocommit = 0;


# 3. 开始事务
start transaction; / begin;


# 4. 执行 SQL 操作
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;


# 5. 提交事务 / 回滚事务
# 5.1. 提交事务（将事务中的所有操作永久保存到数据库中）
commit;

# 5.2. 回滚事务（撤销事务中的所有操作）
rollback;
```

> [!NOTE] 注意事项
> 1. 在 SQL 中，`commit` 和 `rollback` 是互斥的，一个事务只能执行其中一个
> 2. 我们通常是使用编程语言来控制的，成功了就提交，不成功就回滚

---


#### 5.5. 事务隔离性问题

以下这三种现象看似与 **查询操作** 相关，但它们本质上是由于 **事务隔离性** 不足，导致**并发事务**之间在数据访问和修改时发生 **冲突**。
1. ==脏读==：
	1. 假设事务 A 正在对某些数据进行增删改操作，但尚未提交，事务 B 需要读取这些数据。如果事务 B 正好读取了事务 A 正在修改的数据，则事务 B 读取到的数据就是 "脏" 的。
	2. 这些数据被称为“脏数据”，因为事务 A 可能在后续回滚或再次修改，导致事务 B 读取到的数据实际上未被最终提交，可能会导致错误的业务逻辑，尤其是事务 B 后续依赖这些数据时。
2. ==不可重复读==：
	1. 发生在 **同一事务** 内 **多次查询** **同一行数据** 时。
	2. 例如，事务 B 初次查询某行数据，然后事务 A 对这行数据进行了**修改**或**删除**。如果事务 B 再次查询同一行数据，会发现读取到的数据发生了变化，导致 **不可重复读**。
3. ==幻读==：
	1. 发生在 **同一事务** 内 **多次查询** **结果集** 时
	2. 例如，事务 B 查询某个条件范围的数据（如 `age > 50`），然后事务 A 向该范围内**插入**数据。事务 B 再次查询时，发现数据集发生了变化，查询结果与第一次查询不同，这就是 **幻读**。（只有插入才会导致幻读，更新、删除不会导致幻读）

---


#### 5.6. 事务隔离级别

##### 5.6.1. READ UNCOMMITTED（读未提交）

1. ==执行 查询 语句==：
	1. 查询语句本身不会对数据加锁，因此其他事务仍然可以对相关数据执行增删改查操作，可能导致 **不可重复读** 和 **幻读**
	2. 如果查询的数据正好被其他事务修改，则会发生发生 **脏读**
	3. 简单来说：查询有 脏读、不可重复读、幻读 问题
2. ==执行 增、删、改 语句==：
	1. 对操作的数据，会加上行级排他锁（X锁），防止其他事务对这些数据进行增删改操作（可以读取数据，具体是进行脏读还是读取已提交的版本，取决于它的事务的隔离级别）。X锁会一直保持，直到事务结束（提交或回滚）时释放
	2. 如果数据正在被其他事务操作且已加锁，会产生锁冲突，当前事务会等待对方事务结束（提交或回滚），然后再加锁并执行操作。
	3. 如果数据未被锁定，则可以直接进行增删改操作。
3. ==使用场景==:
	1. 数据一致性要求不高，性能最高

---


##### 5.6.2. READ COMMITTED（读已提交，默认）

1. ==执行 查询 语句==：
    1. 查询语句**不会对数据加锁**，因此其他事务仍然可以对相关数据执行增删改查操作，可能导致 **不可重复读** 和 **幻读**
    2. 查询语句**仅读取其他事务已提交的最新数据版本**（**避免脏读**，不会等待提交，直接找最近提交的）。
    3. 简单来说：查询有 不可重复读、幻读 问题
2. ==执行 增、删、改 语句==：
	1. 对操作的数据，会加上**行级排他锁（X锁）**，防止其他事务对这些数据进行增删改操作（可以读取数据，具体是进行脏读还是读取已提交的版本，取决于它的事务的隔离级别）。X锁会一直保持，直到事务结束（提交或回滚）时释放
	2. 如果数据正在被其他事务操作且已加锁，会产生锁冲突，当前事务会等待对方事务结束（提交或回滚），然后再加锁并执行操作。
	3. 如果数据未被锁定，则可以直接进行增删改操作。
3. ==使用场景==：
	1. 数据库的默认隔离级别，日常用这个就行，不可重复读、幻读这些都是小问题，谁还真一次事务执行多次查询啊

---


##### 5.6.3. REPEATABLE READ（可重复读）

1. ==执行 查询 语句==：
	1. 查询语句虽然**不会对数据加锁**，因此其他事务仍然可以对相关数据执行增删改查操作
	2. **但会**在首次查询时创建一个 **一致性数据快照**。后续的查询将基于此快照进行，确保数据一致性。
		1. 如果其他事务对数据进行删除或修改操作，这些变动不会影响快照的内容。因此，后续查询结果与第一次查询一致，避免了**不可重复读**。
		2. 然而，**幻读**仍然可能发生。若其他事务插入了符合查询条件的新记录（例如，`age > 25`），那么在事务 A 下一次查询时，新增的记录会出现在查询结果中，导致两次查询结果不同，从而产生**幻读**。
	3. 查询语句**仅读取其他事务已提交的最新数据版本**（**避免脏读**，不会等待提交，直接找最近提交的）。
	4. 简单来说：查询有 幻读 问题
2. ==执行 增、删、该 语句==：
	1. 对操作数据加**行级排他锁（X锁）**，并对**数据范围**的 **“间隙” 加锁**（例如，当 `WHERE id > 100` 时，锁定 `id > 100` 的区间，注意是区间，不是行数据），以阻止其他事务插入或修改该区间的数据（可以读取数据，具体是进行脏读还是读取已提交的版本，取决于它的事务的隔离级别）
	2. 如果数据正在被其他事务操作且已加锁，会产生锁冲突，当前事务会等待对方事务结束（提交或回滚），然后再加锁并执行操作。
	3. 如果数据未被锁定，则可以直接进行增删改操作。
3. ==使用场景==：
	1. 一致性较高

----


##### 5.6.4. SERIALIZABLE（可串行化）

1. ==执行 查询 语句==：
	1. 查询语句会对数据加 共享锁（S锁），并对**数据范围**的 **“间隙” 加锁**（例如，当 `WHERE id > 100` 时，锁定 `id > 100` 的区间，注意是区间，不是行数据）以阻止其他事务插入或修改该区间的数据（可以读取数据，具体是进行脏读还是读取已提交的版本，取决于它的事务的隔离级别）
	2. 若使用索引查询，锁定匹配的索引范围和间隙
	3. 若全表扫描，可能直接加表级锁
	4. 简单来说：查询无 脏读、不可重复度、幻读 问题
2. ==执行 增、删、该 语句==：
	1. 对操作数据加**行级排他锁（X锁）**，并对**数据范围**的 **“间隙” 加锁**（例如，当 `WHERE id > 100` 时，锁定 `id > 100` 的区间，注意是区间，不是行数据），以阻止其他事务插入或修改该区间的数据（可以读取数据，具体是进行脏读还是读取已提交的版本，取决于它的事务的隔离级别）
	2. 如果数据正在被其他事务操作且已加锁，会产生锁冲突，当前事务会等待对方事务结束（提交或回滚），然后再加锁并执行操作。
	3. 如果数据未被锁定，则可以直接进行增删改操作。
3. ==使用场景==：
	1. 一致性最高，性能最低

---


### 6. 数据分区

---


### 7. MySQL 约束

#### 7.1. 约束概述

约束是数据库中用于限制和控制**列上数据**完整性的一种规则。

我的策略是：除了主键约束，其他约束都在表创建后再添加约束。

---


#### 7.2. 常用约束

==1.主键约束==

| 约束类型     | 关键字              | 说明                                                                   |
| -------- | ---------------- | -------------------------------------------------------------------- |
| **主键约束** | `PRIMARY KEY`    | 用于确保列中的每个**值唯一**且**不能为空**，通常主键列还会**设置自增属性**以便自动生成唯一值。每个表只能有一个主键。<br> |
| **自增属性** | `AUTO_INCREMENT` | 不是传统意义上的约束，而是一种属性，用于为整数类型的列设置自增功能，自动为该列生成唯一的数字值。**通常用于主键列。**<br>     |
```
# 1. 主键约束
# 1.1. 添加主键约束
alter table <table-name> add constraint <constraint-name> primary key (<column-name>);

# 1.2. 删除主键约束（自动删除唯一索引）
alter table <table-name> drop primary key;


# 2. 自增属性
# 2.1. 添加自增属性
alter table <table-name> modify <column-name> <data-type> auto_increment;

# 2.2. 删除自增属性
alter table <table-name> modify <column-name> <data-type>;
```

> [!NOTE] 注意事项：
> 1. 主键约束会自动添加 **非空约束** 和 **唯一约束**，并创建 **主键索引**（该索引本身也是唯一索引的一种形式，因此保证了每个值的唯一性）。通常情况下，主键列还会设置为 **自增** 属性。
> 2. 自增属性仅适用于**数值类型**的列
> 3. `constraint-name` 是我们为约束起的约束名称，方便后续维护和开发
> 4. `data-type` 中要写具体的数值类型，例如是 `int` 还是 `bigint`


==2.外键约束==

| 约束类型         | 关键字               | 说明                  |
| ------------ | ----------------- | ------------------- |
| **外键约束**<br> | `FOREIGN KEY`<br> | 让两张表的列中数据之间产生连接<br> |
```
# 1. 添加外键约束
alter table <table-name> add constraint <constraint-name> foreign key (<column-name>) references <referenced-table> (<referenced-column>) [ON UPDATE 更新行为] [ON DELETE 删除行为];


# 2. 删除外键约束
alter table <table-name> drop foreign key <constraint-name>;
```


==3.普通约束==

| **约束类型**  | **关键字**    | **说明**                                   |
| --------- | ---------- | ---------------------------------------- |
| **唯一约束**  | `UNIQUE`   | 用于确保列中的值**在表中是唯一**的，允许 `NULL` 值。         |
| **检查约束**  | `CHECK`    | 用于确保列中的值满足特定的条件或表达式。在 MySQL 8.0 及以上版本支持。 |
| **非空约束**  | `NOT NULL` | 用于确保列中的值不能为 `NULL`。                      |
| **默认值约束** | `DEFAULT`  | 用于为列设置默认值，如果插入数据时没有指定该列的值，数据库将自动使用默认值。   |

```
# 1. 唯一约束（☆☆☆）
# 1.1. 添加唯一约束
alter table <table-name> add constraint <constraint-name> unique (<column-name>);

# 1.2. 删除唯一约束
alter table <table-name> drop index <constraint-name>;


# 2. 检查约束
# 2.1. 添加检查约束
alter table <table-name> add constraint <constraint-name> check (<condition>);
alter table <table-name> add constraint <constraint-name> check (<column1-name> > 0 and <column2-name> < 100);

# 2.2. 删除检查约束
alter table <table-name> drop check <constraint-name>;


# 3. 非空约束
# 3.1. 添加非空约束
alter table <table-name> modify <column-name> <data-type> not null;

# 3.2. 删除非空约束
alter table <table-name>modify <column-name> <data-type> null;


# 4. 默认值约束
# 4.1. 添加默认值约束
alter table <table-name> modify <column_name> <data_type> default <default_value>;

# 4.2. 删除默认值约束
allter table <table-name> modify <column-name> <data_type>
```

> [!NOTE] 注意事项
> 1. 特别注意 **唯一约束**，因为唯一约束会自动创建 **唯一索引**，因此命令稍有不同。其他三种约束命令上差异不大。
> 2. 要修改约束，我们先删除旧约束，再添加新约束

----


#### 7.3. 补充：外键约束


##### 7.3.1. 外键约束建立原则

外键约束**定义在从表**中，用于**引用主表的主键或唯一键**。

---


##### 7.3.2. 一对一关系

在一对一关系中，两个表中的每一行记录都与另一个表中的一行记录相关联。例如，一个员工对应一个工作证，每个工作证对应一个员工。

在一对一关系中，通常选择**列较少的表作为主表**，**列较多的表作为从表**，并在主表上建立外键。

以 `Employee` 表和 `WorkCard` 表为例，`Employee` 表中的每个员工都有一个唯一的 `EmpID`，而 `WorkCard` 表中的每个工作证都对应一个唯一的员工。
```
# 1. Employee 表（主表）
CREATE TABLE Employee (
    EmpID INT PRIMARY KEY,
    Name VARCHAR(100)
);


# 2. WorkCard 表（从表）
CREATE TABLE WorkCard (
    CardID INT PRIMARY KEY,
    EmpID INT, // 从表的外键字段 -> 主表的字段（一般命名相同）
    IssueDate DATE,
    FOREIGN KEY (EmpID) REFERENCES Employee(EmpID)
);
```
![](source/_posts/笔记：MySQL%20基础/image-20250521211043750.png)

![](source/_posts/笔记：MySQL%20基础/image-20250521211108505.png)

> [!NOTE] 注意事项：
> 1. 删除数据时，先删除从表数据，再删除主表数据
> 2. 插入数据时，先插入主表数据，再插入从表
```
# 1. 先插入 Employee 表（主表）数据
INSERT INTO Employee (EmpID, Name) 
VALUES (1, 'John Doe');


# 2. 再插入 WorkCard 表（从表）数据
INSERT INTO WorkCard (CardID, EmpID, IssueDate)
VALUES (101, 1, '2025-04-09');
```

---


##### 7.3.3. 一对多

在一对多关系中，一个表中的一行记录可以与另一个表中的多行记录相关联。常见的例子是：一个部门有多个员工（多），而每个员工只能属于一个部门（一）。

在一对多关系总，选择 **“一”的表作为主表**，**“多”的表作为从表**

假设有 `Department`（部门表）和 `Employee`（员工表）。一个部门有多个员工，每个员工属于一个部门。
```
# 1. Department 表（主表）
CREATE TABLE Department (
    DeptID INT PRIMARY KEY,
    DeptName VARCHAR(100)
);


# 2. Employee 表（从表）
CREATE TABLE Employee (
    EmpID INT PRIMARY KEY,
    Name VARCHAR(100),
    DeptID INT,
    FOREIGN KEY (DeptID) REFERENCES Department(DeptID)
);
```
![](source/_posts/笔记：MySQL%20基础/image-20250521212132677.png)

![](source/_posts/笔记：MySQL%20基础/image-20250521212154361.png)

> [!NOTE] 注意事项：
> 1. 删除数据时，先删除从表数据，再删除主表数据
> 2. 插入数据时，先插入主表数据，再插入从表

---


##### 7.3.4. 多对多

在多对多关系中，表中的多行记录与另一个表中的多行记录相互关联。常见的例子是：一个学生可以选修多门课程（多），每门课程也可以有多个学生选修（多）。

在多对多关系中，我们通常需要 **一个中间表** 来存储关联信息。**在这个中间表中**，我们为每一对关联的表**添加外键**。

假设有 `Student`（学生表）和 `Course`（课程表），一个学生可以选修多门课程，而每门课程可以有多个学生选修。我们需要一个中间表 `Enrollment` 来记录学生与课程之间的关系。
```
# 1. Student 表（主表）
CREATE TABLE Student (
    StudentID INT PRIMARY KEY,
    Name VARCHAR(100)
);


# 2. Course 表（主表）
CREATE TABLE Course (
    CourseID INT PRIMARY KEY,
    CourseName VARCHAR(100)
);


# 3. Enrollment 表（从表）
CREATE TABLE Enrollment (
    StudentID INT,
    CourseID INT,
    PRIMARY KEY (StudentID, CourseID),
    FOREIGN KEY (StudentID) REFERENCES Student(StudentID),
    FOREIGN KEY (CourseID) REFERENCES Course(CourseID)
);
```

> [!NOTE] 注意事项
> 1. 据库设计推荐在这种“纯粹中间表”里省略 id 字段，因为我们PRIMARY KEY (StudentID, CourseID),作为**联合主键**，这已经能唯一标识每一行数据了。
> 2. 删除数据时，先删除从表数据，再删除主表数据（先删除中间表数据）
> 3. 插入数据时，先插入主表数据，再插入从表（先左再右后中间）
> 4. 需要插三次，先左再右后中间

---


##### 7.3.5. 外键约束下的删除与更新行为

|**行为**|**说明**|
|---|---|
|**CASCADE**|如果主表中的数据被删除或更新，外键表中的相关数据会被自动删除或更新。|
|**SET NULL**|如果主表中的数据被删除或更新，外键表中相关数据的外键列会被设置为 `NULL`。|
|**NO ACTION**|如果主表中的数据被删除或更新，外键约束会阻止该操作发生。如果外键表中存在与主表相关的数据，则不能执行删除或更新操作。|
|**RESTRICT**|与 `NO ACTION` 类似，阻止删除或更新操作。如果外键表中有相关数据，删除或更新会被拒绝。|
|**SET DEFAULT**|如果主表中的数据被删除或更新，外键表中的相关外键列会被设置为该列的默认值（需要列定义默认值）。|

---


### 8. MySQL 索引

#### 8.1. 索引存储形式

在 `InnoDB` 存储引擎中，根据索引的存储形式，可以分为聚集索引和二级索引：
1. ==聚集索引==：
	1.一个表中**只能有且必须有一个**聚集索引。
	1. 聚集索引将数据和索引存储在一起，索引结构的叶子节点直接保存行数据
	2. 聚集索引的选取规则：
		1. 如果存在**主键索引**，主键索引即为聚集索引
		2. 如果没有主键索引，第一个**唯一索引**会作为聚集索引
		3. 如果既没有主键也没有唯一索引，`InnoDB` 会自动为表生成一个**隐藏**的 `rowid` 作为聚集索引
2. ==二级索引==：
	1. 一个表中可以存在多个二级索引
	2. 二级索引将数据和索引分开存储，索引结构的叶子节点存储的是对应主键的值。
	3. 在 `InnoDB` 存储引擎中，当你查询二级索引列的数据时，过程如下：
		1. 查询会首先通过二级索引查找该列的值，并返回二级索引中的存储的主键值
		2. 得到主键值后，查询会使用主键值回到聚集索引中查找实际的数据行。
![](source/_posts/笔记：MySQL%20基础/image-20250409183911323.png)

![](source/_posts/笔记：MySQL%20基础/image-20250409192001892.png)

---


#### 8.2. 索引的分类

| **索引类型** | **关键字**         | **描述**                             | **应用场景**                                      |
| -------- | --------------- | ---------------------------------- | --------------------------------------------- |
| **主键索引** | `PRIMARY KEY`   | 每个表只能有一个主键索引，主键列不允许有 NULL 值。       | 用于唯一标识表中的每一行，通常是表的主键（例如，`id` 列）。              |
| **唯一索引** | `UNIQUE`        | 保证索引列的值是唯一的，但可以有 NULL 值。           | 用于确保数据的唯一性，适用于需要保持唯一性的字段（如邮箱地址、用户名等）。         |
| **普通索引** | `INDEX` 或 `KEY` | 普通索引是最基本的索引类型，不保证值唯一。              | 用于提高查询速度，适用于不要求唯一性但需要快速查找的字段。                 |
| **全文索引** | `FULLTEXT`      | 用于全文搜索，通常在 `TEXT` 类型的列上使用，支持单词的查找。 | 用于执行全文搜索，如在文章或评论内容中查找关键词。（直接上手 ElasticSearch） |
| **复合索引** | `INDEX` 或 `KEY` | 由多个列组成的索引。                         | 当查询涉及多个字段时，可以使用复合索引来提高查询性能，避免多个单列索引的使用。       |

> [!NOTE] 注意事项
> 1. 一列可以同时有多个不同类型的索引（例如普通索引 + 唯一索引 + 全文索引），但不能有多个相同类型的索引（如两个唯一索引）。

---


#### 8.3. 索引相关命令


```
# 1. 添加索引
# 1.1. 主键索引
通过创建主键约束来自动生成主键索引。

# 1.2. 唯一索引
create unique index <index-name> on <table-name> (<column-name>);

# 1.3. 普通索引
create index <index-name> on <table-name> (<column-name>);

# 1.4. 全文索引
create fulltext index <index-name> on <table-name> (<column-name>);

# 1.5. 复合索引
create index <index-name> on <table-name> (<column1>, <column2>, ...);


# 2. 删除索引
drop index <index-name> on <table-name>;


# 3. 查询索引
show index from <table-name>;
```

> [!NOTE] 注意事项
> 1. `<index-name>` 是为索引自定义的名称，主要用于后续的维护、查看和删除操作
> 2. 唯一索引和唯一约束效果相同，只是语义上的差异：
> 	- <font color="#00b0f0">唯一索引</font>：
> 		- 更偏向结构设计的“规范性”
> 	- <font color="#00b0f0">唯一约束</font>：
> 		- 更偏向数据库“性能调优”的语义表达

---


### 9. MySQL 视图

#### 9.1. 视图概述

视图是一个虚拟的表，它**内部定义了 SQL 查询语句**，每次你查询视图时，数据库会执行视图中的查询，并动态生成结果，你可以理解为，视图是将一个 SQL 查询的结果作为一个虚拟表进行保存。视图允许用户在不直接访问底层表的情况下，通过查询视图来获取数据。

简单来说，视图充当了数据的抽象层，提供了一个接口，使得客户端可以直接查询视图，而无需关心底层表的结构。可以将查询路径理解为：客户端 -> 视图 -> 表。使用视图的好处有很多：
1. ==提高数据安全==：
	1. 视图可以限制客户端对某些敏感数据的访问。例如，如果你有一个包含用户个人信息的表，你可以创建一个不包含敏感信息（如密码、身份证号等）的视图，来对外提供查询。
	2. 作为客户端开发者，你自然不需要限制自己的访问权限，但如果需要简化查询逻辑，视图仍然是一个方便的工具
	3. 但是在多用户环境下，你可以通过视图限制用户仅访问他们有权限查看的数据，确保敏感信息的安全
2. ==简化查询逻辑==：
	1. 假设有一个复杂的查询，涉及多个表、连接、筛选和排序操作。你可以将这些操作封装成一个视图，之后只需查询视图，避免每次都编写复杂的 SQL 查询。
```
# 1. 复杂的查询逻辑
CREATE VIEW employee_summary AS
SELECT emp.id, emp.name, dep.name AS department, COUNT(proj.id) AS project_count
FROM employees emp
JOIN departments dep ON emp.department_id = dep.id
LEFT JOIN projects proj ON emp.id = proj.employee_id
GROUP BY emp.id;


# 2. 直接查询视图
SELECT * FROM employee_summary WHERE department = 'IT';
```

> [!NOTE] 注意事项
> 1. 在某些特定情况下，**视图可以进行更新**（即可以通过视图执行 `INSERT`、`UPDATE` 或 `DELETE` 操作），但通常情况下，视图**主要用于查询**，**不建议用于数据更新操作**

---


#### 9.2. 视图相关命令
如果需要修改视图的定义，可以使用 `CREATE OR REPLACE VIEW` 来替换已有的视图。注意，`REPLACE` 会删除旧视图并创建新的视图。
```
# 1. 创建视图
create view [if not exists] <view-name> as select <column1>, <column2>, ... from <table-name> <where condition>;


# 2. 删除视图
drop view [if exists] <view-name>;


# 3. 更新视图（相当于重新写了一个视图，会删除旧视图，新建新视图）
create or replace view <view-name> as select <new-column1>,<new-column2>, ... from <table-name> where <your-new-ondition>;


# 4. 查询视图（正常怎么查询，这里怎么查询）
select * from view_name;
```

---


### 10. MySQL 锁

---


### 11. MySQL 调优

#### 11.1. SQL 执行频率

通过分析 SQL 执行的比例，我们可以判断数据库或表的主要操作类型是查询（SELECT）还是增删改（INSERT、UPDATE、DELETE）。如果主要是查询操作，我们可以重点进行索引优化，以提升查询性能。
```
# 1. 查看全局数据
show global status like 'com%';


# 2. 查看当前会话数据
show session status like 'com%';
```

> [!NOTE] 注意事项
> 1. 给出的 `value` 是次数，`value` 多大就代表有多少次


---


#### 11.2. 慢查询日志

慢日志记录了所有执行时间超过指定参数（`long_query_time`，单位：秒，默认10秒）的所有SQL语句的日志。
```
# 1. 查看慢日志是否开启
show variables like 'slow_query_log';


# 2. 开启慢日志
# 2.1. 编辑配置文件
vim /data/mysql/etc/mysql/conf.d/my.cnf                       # 卷挂载

# 2.2. 添加以下内容
[mysqld]                                                      # mysql 区块
slow_query_log = 1                                            # 开启慢日志
long_query_time = 0.2                                         # 设置慢日志的阈值（单位：秒）
slow_query_log_file = /var/log/mysql/mysql-slow.log           # 设置慢日志的保存路径
log_queries_not_using_indexes = 1                             # 把所有没有使用索引的 SQL 查询也记录到慢查询日志中，即使它们没超过 long_query_time 阈值


# 3. 重启容器
docker restart <container-name> / <container-id>


# 4. 查询慢日志
cat /var/lib/mysql/localhost-slow.log
```

> [!NOTE] 注意事项
> 1. 只要你是在 `conf.d`、`mysql.conf.d`、或者自定义的 `.cnf` 文件里配置，请务必写上 `[mysqld]`
> 2. 开启慢查询日志后，当你执行 SQL 语句时，如果查询执行时间超过设定的阈值，MySQL 会自动记录该查询为慢查询，并且会立即提醒：

![](source/_posts/笔记：MySQL%20基础/image-20250409223121020.png)

---


#### 11.3. Profile 查看详情

Profiles 可以帮助我们分析时间的分配，明确哪些环节消耗了最多的时间。
```
# 1. 查看 MySQL 是否支持 Profile
select @@have_profiling;


# 2. 开启 Profile
set profiling = 1;


# 3. 执行一系列 SQL 语句
select * from characters where id=1;
select * from characters where id=2;
select * from characters where id=3;


# 4. 查看 SQL 语句耗时时间
show profiles;


# 5. 根据 query-id 查看 SQL 语句每个阶段的耗时情况
show profile for query <query-id>;


# 6. 根据 query-id 查看 SQL 语句 CPU 使用情况
show profile cpu for query <query-id>;
```

---


#### 11.4. Explain

`explain` 命令用于获取 mysql 执行 `select` 语句的执行计划信息，包括表如何连接、连接顺序等细节。
```
explain <select 语句>
```
![](source/_posts/笔记：MySQL%20基础/image-20250410100653127.png)
1. ==id==:
	1. 查询的 **标识符**，它表示在多表查询中每个操作的执行顺序。例如，`id = 1` 是最先执行的操作，`id = 2` 是第二个执行的操作，依此类推。
	2. 在单表查询中，`id` 始终为 `1`。
	3. 在多表查询中，`id` 会被分配一个递增的数字，用来标识查询的执行顺序。
2. ==select_type==：
	1. 查询的类型
	2. <font color="#00b0f0">SIMPLE</font>：
		1. 简单的查询，没有使用子查询或联合查询。
	3. <font color="#00b0f0">PRIMAR</font>：
		1. 表示主查询（最外层查询），如果查询中有子查询，主查询会标记为 `PRIMARY`。
	4. <font color="#00b0f0">SUBQUER</font>：
		1. 表示子查询。
	5. <font color="#00b0f0">DEPENDENT SUBQUER</font>：
		1. 依赖于外部查询的子查询。
	6. <font color="#00b0f0">UNION</font>：
		1. 表示 `UNION` 查询中的第二个查询。
3. ==table==：
	1. 查询的 **表**，表示该行对应的操作是针对哪张表进行的。
4. ==partitions==：
	1. 查询操作作用的 **分区**。
	2. 在使用分区表时，表示该查询在哪些分区上执行。
	3. 没有分区的表，这一列会显示为 `NULL`。
5. ==type==：
	1. 连接类型，表示 MySQL 查询优化器选择的访问方式。一下是性能由差到好的顺序：
	2. <font color="#00b0f0">all</font>：
		1. 全表扫描，表示 MySQL 会扫描表中的所有记录，效率较低。
	3. <font color="#00b0f0">index</font>：
		1. 索引扫描，表示 MySQL 会扫描整个索引而不是整个表。
	4. <font color="#00b0f0">range</font>：
		1. 范围扫描，表示 MySQL 会使用索引查找某个范围的记录。
	5. <font color="#00b0f0">ref</font>：
		1. 基于非唯一索引的扫描。
	6. <font color="#00b0f0">eq_ref</font>：
		1. 基于唯一索引的扫描，每个记录都能对应唯一的索引值。
	7. <font color="#00b0f0">const</font>：
		1. 常量查找，表示表中的每一行记录都可以通过常量来匹配，效率非常高。
	8. <font color="#00b0f0">system</font>：
		1. 表示系统表，通常表示只有一行数据的表（如数据库的 `information_schema` 表）。
6. ==possible_keys==：
	1. 本次查询可能使用到的索引，如果没有使用索引，则为 null
7. ==key==：
	1. 实际使用的 **索引**，表示查询中实际使用的索引。
	2. 如果该字段显示为 `NULL`，表示没有使用索引，可能进行了全表扫描
	3. 如果显示了索引名，表示优化器选择了这个索引来执行查询
8. ==key_len==：
	1. 表示优化器使用的索引的 **长度**（单位：字节）。该值通常与索引列的类型和数量相关。
	2. 在不损失精确性的前提下， 长度越短越好 。
9. ==ref==：
	1. **索引列的比较值**，显示该查询使用了哪个字段（或常量）来与索引中的值进行比较。
10. ==rows==：
	1. 估计 **需要扫描的行数**，表示查询执行时，优化器估算的需要扫描的表中记录的数量。
11. ==filtered==：
	1. 表示返回结果的行数占需读取行数的百分比， filtered 的值越大越好。
12. ==Extra==：
	1. 提供有关查询执行的额外信息

















# 二、实操（搭建 MySQL）


### 1. 环境搭建

#### 1.1. 单机测试环境搭建

==1.创建宿主机数据挂载目录==
```
mkdir -p /mystudy/data/mysql
```


==2.启动 MySQL 容器==
```
docker run -d \
  --name my_mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=wq666666 \
  -v /mystudy/data/mysql:/var/lib/mysql \
  mysql:8.0 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci
```

---


### 分布式集群环境搭建


















### 2. 高可用集群（非 K8S）




---



### 3. 高可以集群（K8S）


# 三、工具

### 1. MyCat2



---



















### 2. Java Faker

```
<dependencies>
    <dependency>
        <groupId>com.github.javafaker</groupId>
        <artifactId>javafaker</artifactId>
        <version>1.0.2</version>
    </dependency>
</dependencies>
```


```
import com.github.javafaker.Faker;

import java.util.Locale;

public class FakerDemo {
    public static void main(String[] args) {
        // 创建 Faker 实例，设置为中文（zh-CN）
        Faker faker = new Faker(new Locale("zh-CN"));

        // 基本信息
        String name = faker.name().fullName();        // 中文全名
        String email = faker.internet().emailAddress();
        String phone = faker.phoneNumber().cellPhone();
        String address = faker.address().fullAddress();

        // 职业、公司、行业
        String job = faker.job().title();
        String company = faker.company().name();
        String industry = faker.company().industry();

        // 黑客/技术术语（需要英文环境）
        Faker enFaker = new Faker();  // 默认英文
        String techPhrase = enFaker.hacker().phrase();

        // 输出
        System.out.println("👤 姓名：" + name);
        System.out.println("📧 邮箱：" + email);
        System.out.println("📞 手机：" + phone);
        System.out.println("🏠 地址：" + address);
        System.out.println("💼 职位：" + job);
        System.out.println("🏢 公司：" + company);
        System.out.println("🏭 行业：" + industry);
        System.out.println("💻 黑客术语：" + techPhrase);
    }
}
```


| 模块              | 方法                              | 示例内容                                            |
| --------------- | ------------------------------- | ----------------------------------------------- |
| `name()`        | `.fullName()`                   | 张伟、李娜                                           |
| `internet()`    | `.emailAddress()`               | [abc123@example.com](mailto:abc123@example.com) |
| `phoneNumber()` | `.cellPhone()`                  | 139-8888-7777                                   |
| `address()`     | `.fullAddress()`                | 北京市朝阳区某路100号                                    |
| `job()`         | `.title()`                      | 数据工程师、UI设计师                                     |
| `company()`     | `.name()` `.industry()`         | 腾讯、人工智能行业                                       |
| `hacker()`      | `.verb()` `.noun()` `.phrase()` | bypass the firewall...                          |
| `lorem()`       | `.paragraph()`                  | 模拟一段看起来有逻辑的文本                                   |


### 3. DataGrip


#### 3.1. 汉化

![](source/_posts/笔记：MySQL%20基础/image-20250519144517641.png)

---













# 二、实操


### 1. 安装 MyCat2

==1.安装 JDK==
使用MyCAT 2要安装JDK，因为MyCAT 是基于JDK1.8开发的
```
sudo apt install openjdk-17-jdk
```


==2.创建存放 MyCat 的目录==
```
# 1. 常见 /tools
mkdir -p /tools


# 2. 进入 /tools
cd /tools
```


==3.将 MyCat2 相关包拖过来==
将 `mycat2-install-template-1.21.zip` 和 `mycat2-1.22-release-jar-with-dependencies.jar` 拖到 `/tools` 目录下


==4.解压 MyCat2 安装包==
```
# 1. 安装 unzip 工具（需要解压 .zip 包）
sudo apt install unzip


# 2. 解压 MyCat2 安装包
unzip mycat2-install-template-1.21.zip


# 3. 增加对 /tools/mycat/bin 的执行权限
cd /tools/mycat/bin

chmod +x *
```


==5.复制 jar 包到 /tools/mycat/lib 目录==
```
cp /tools/mycat2-1.22-release-jar-with-dependencies.jar /tools/mycat/lib
```





绕过 Linux 的用户名和密码，直接使用 MyCat2 的用户名和密码进行登录
![](source/_posts/笔记：MySQL%20基础/image-20250523113818549.png)


启动有问题看日志：
```
cat /tools/mycat/logs/wrapper.log
```

---


### 2. 安全管理










# 三、补充

### 1. MyCat2 的目录结构

#### 1.1. 目录总览

![](source/_posts/笔记：MyCat2/image-20250406155610051.png)
1. ==bin==：
	1. 执行主要命令的目录
2. ==conf==：
	1. 软件的配置文件
3. ==lib==：
	1. 该软件的依赖包
4. ==logs==：
	1. 该软件的日志包

---




























# 四、补充

### 1. 相关网站

1. MyCat2 官方网站：
	1. http://mycatone.top/

### 2. MySQL 目录结构

```
根目录 /
|
|-- var /
|   |
|   |-- lib /
|   |   |
|   |   |-- mysql /                               # 默认的数据目录       
|   |       |
|   |       |-- ibdata1                           # InnoDB 的共享表空间
|   |       |
|   |       |-- mysql /                           # 存放 MySQL 用户、权限、时区等系统信息
|   |       |
|   |       |-- yout_database_name /              # 你的库数据和相关信息，一个库一个目录
|   |
|   |-- log /                                     # 存放日志（错误日志、慢查询日志等）
|
|-- etc /
|   |
|   |-- my.cnf                                    # 全局配置文件，定义 MySQL 基础参数
|   |
|   |-- my.cnf.d /                                # 存放独立配置文件，可以覆盖或扩展主配置
|   | 
|   |-- mysql /
|   |   |
|   |   |-- conf.d /                              # 存放独立配置文件，可以覆盖或扩展配置文件
```

> [!NOTE] 注意事项：配置文件的优先级
> 1. `/etc/my.cnf.d/*.cnf` > `/etc/mysql/conf.d/*.cnf` > `my.cnf`

---


### 3. 常用数据类型

==1.数值类型==

|名称|字节数|描述|
|---|---|---|
|`TINYINT`|1|很小的整数，范围：有符号 `-128~127`，无符号 `0~255`|
|`SMALLINT`|2|小整数，有符号 `-32,768~32,767`，无符号 `0~65,535`|
|`MEDIUMINT`|3|中等整数，有符号 `-8,388,608~8,388,607`，无符号 `0~16,777,215`|
|`INT` 或 `INTEGER`|4|标准整数，有符号 `-2^31~2^31-1`，无符号 `0~2^32-1`|
|`BIGINT`|8|大整数，有符号 `-2^63~2^63-1`，无符号 `0~2^64-1`|
|`FLOAT(M,D)`|4|单精度浮点数，M 是总位数，D 是小数位数（精度约为 7 位十进制）|
|`DOUBLE(M,D)` 或 `REAL`|8|双精度浮点数（精度约为 15 位十进制）|
|`DECIMAL(M,D)` 或 `NUMERIC(M,D)`|变长|精确小数，不丢失精度，M 是总位数，D 是小数位（用于金融等）|


==2.字符串类型==

|名称|字节数|描述|
|---|---|---|
|`CHAR(n)`|固定 n 字节（1 ≤ n ≤ 255）|固定长度字符串，不足部分补空格，适合长度一致的数据|
|`VARCHAR(n)`|实际长度 + 1/2 字节|可变长度字符串，n ≤ 65,535（受行大小和字符集影响）|
|`TEXT`|最大 65,535 字节（64KB）|大文本字段，不可创建索引（除非指定长度）|
|`TINYTEXT`|最大 255 字节|小文本字段|
|`MEDIUMTEXT`|最大 16,777,215 字节（16MB）|中文本字段|
|`LONGTEXT`|最大 4,294,967,295 字节（4GB）|大文本字段|
|`BINARY(n)`|固定长度二进制数据|类似 CHAR，但存储二进制数据|
|`VARBINARY(n)`|可变长度二进制数据|类似 VARCHAR，但存储二进制数据|
|`BLOB`|二进制大对象，最大 64KB|用于存储二进制数据，如图片、音频|
|`TINYBLOB`|最大 255 字节|小二进制对象|
|`MEDIUMBLOB`|最大 16MB|中等大小二进制对象|
|`LONGBLOB`|最大 4GB|超大二进制对象|
|`ENUM`|1 或 2 字节|枚举类型，取值需从预设的字符串中选（如 `'male', 'female'`）|
|`SET`|1~8 字节|多选枚举（可同时选多个值）|


==3.时间和日期类型==

|名称|字节数|描述|格式示例|
|---|---|---|---|
|`DATE`|3|仅日期，范围：`1000-01-01 ~ 9999-12-31`|`'YYYY-MM-DD'` → `2025-04-09`|
|`TIME`|3|仅时间，范围：`-838:59:59 ~ 838:59:59`|`'HH:MM:SS'` → `14:23:59`|
|`DATETIME`|8|日期 + 时间，范围：`1000-01-01 00:00:00` ~ `9999-12-31 23:59:59`|`'YYYY-MM-DD HH:MM:SS'` → `2025-04-09 14:23:59`|
|`TIMESTAMP`|4|UNIX 时间戳（UTC），1970年起的秒数，可自动更新时间|`'YYYY-MM-DD HH:MM:SS'` → `2025-04-09 06:23:59`|
|`YEAR`|1|表示年份，范围：`1901 ~ 2155`|`'YYYY'` → `2025`|

> [!NOTE] 注意事项： `DATETIME` 和 `TIMESTAMP` 的区别
> 1. <font color="#00b0f0">DATETIME</font>：
> 	- 纯时间点，存储和时区无关
> 1. <font color="#00b0f0">TIMESTAMP</font>
> 	- 是时间戳，会受服务器时区影响（适合记录变动时间）

---


### 4. 常用运算符

==1.比较运算符==

| 比较运算符               | 功能                      | 示例                                                                                              |
| ------------------- | ----------------------- | ----------------------------------------------------------------------------------------------- |
| >                   | 大于                      | SELECT * FROM Employees WHERE Salary > 50000;                                                   |
| >=                  | 大于等于                    | SELECT * FROM Employees WHERE Age >= 30;                                                        |
| <                   | 小于                      | SELECT * FROM Products WHERE Price < 100;                                                       |
| <=                  | 小于等于                    | SELECT * FROM Students WHERE Marks <= 75;                                                       |
| =                   | 等于                      | SELECT * FROM Customers WHERE Country = 'USA';                                                  |
| <> 或 !=             | 不等于                     | SELECT * FROM Orders WHERE Status <> 'Shipped'; SELECT * FROM Orders WHERE Status != 'Shipped'; |
| BETWEEN ... AND ... | 在某个范围之内（含最小、最大值）        | SELECT * FROM Sales WHERE Date BETWEEN '2022-01-01' AND '2022-12-31';                           |
| IN(...)             | 在 in 之后的列表中的值，多选一       | SELECT * FROM Employees WHERE Department IN ('Sales', 'HR', 'IT');                              |
| LIKE 占位符            | 模糊匹配（_匹配单个字符，%匹配任意多个字符） | SELECT * FROM Customers WHERE Name LIKE 'A%';                                                   |
| IS NULL             | 是 NULL                  | SELECT * FROM Employees WHERE ManagerID IS NULL;                                                |


==2.逻辑运算符==

|逻辑运算符|功能|示例|
|---|---|---|
|AND 或 &&|并且（多个条件同时成立）|SELECT * FROM Employees WHERE Age > 25 AND Department = 'IT';|
|OR 或 \||或者（多个条件任意一个成立）|SELECT * FROM Products WHERE Price < 50 OR Stock > 100;|
|NOT 或 !|非，不是|SELECT * FROM Customers WHERE NOT Country = 'USA';|

---


### 5. 常用函数

==1.聚合函数==

| 函数    | 功能   | 举例                                                                               |
| ----- | ---- | -------------------------------------------------------------------------------- |
| count | 统计数量 | SELECT COUNT(*) AS total_rows FROM employees; -- 结果：返回employees表中行的总数            |
| max   | 最大值  | SELECT MAX(salary) AS max_salary FROM employees; -- 结果：返回employees表中salary列的最大值  |
| min   | 最小值  | SELECT MIN(salary) AS min_salary FROM employees; -- 结果：返回employees表中salary列的最小值  |
| avg   | 平均值  | SELECT AVG(salary) AS avg_salary FROM employees; -- 结果：返回employees表中salary列的平均值  |
| sum   | 求和   | SELECT SUM(salary) AS total_salary FROM employees; -- 结果：返回employees表中salary列的总和 |


==2.日期函数==

| 函数                                 | 功能                              | 举例                                                                                     |
| ---------------------------------- | ------------------------------- | -------------------------------------------------------------------------------------- |
| CURDATE()                          | 返回当前日期                          | SELECT CURDATE() AS current_date; -- 结果：例如 '2023-10-10' （根据执行日期不同会有不同的结果）              |
| CURTIME()                          | 返回当前时间                          | SELECT CURTIME() AS current_time; -- 结果：例如 '14:30:45' （根据执行时间不同会有不同的结果）                |
| NOW()                              | 返回当前日期和时间                       | SELECT NOW() AS current_date_time; -- 结果：例如 '2023-10-10 14:30:45' （根据执行日期和时间不同会有不同的结果） |
| YEAR(date)                         | 获取date中的年份                      | SELECT YEAR('2023-10-10') AS year; -- 结果：2023                                          |
| MONTH(date)                        | 获取date中的月份                      | SELECT MONTH('2023-10-10') AS month; -- 结果：10                                          |
| DAY(date)                          | 获取date中的日期                      | SELECT DAY('2023-10-10') AS day; -- 结果：10                                              |
| DATE_ADD(date, INTERVAL expr type) | 返回一个日期/时间值，expr参数加在指定的日期值得到新的日期 | SELECT DATE_ADD('2023-10-10', INTERVAL 5 DAY) AS new_date; -- 结果：'2023-10-15'          |
| DATEDIFF(expr1, expr2)             | 返回起始时间expr1与结束时间expr2之间的天数      | SELECT DATEDIFF('2023-10-15', '2023-10-10') AS diff_days; -- 结果：5                      |
| CURRENT_TIMESTAMP                  | 返回当前的时间，格式是 YYYY-MM-DD HH:MM:SS |                                                                                        |


你这个格式非常清晰，下面我就照你这套格式，补充一些 **常用的 MySQL 日期时间函数**，非常实用：

---

|函数/表达式名|说明|示例|
|---|---|---|
|`CURRENT_TIMESTAMP`|返回当前的时间，格式是 `YYYY-MM-DD HH:MM:SS`|`SELECT CURRENT_TIMESTAMP;` → `2025-05-19 17:45:00`|
|`NOW()`|返回当前时间，等价于 `CURRENT_TIMESTAMP`|`SELECT NOW();` → `2025-05-19 17:45:00`|
|`CURDATE()`|返回当前日期，格式是 `YYYY-MM-DD`|`SELECT CURDATE();` → `2025-05-19`|
|`CURTIME()`|返回当前时间（不含日期），格式是 `HH:MM:SS`|`SELECT CURTIME();` → `17:45:00`|
|`UNIX_TIMESTAMP()`|返回当前的 Unix 时间戳（秒数）|`SELECT UNIX_TIMESTAMP();` → `1747647900`|
|`FROM_UNIXTIME(unix_timestamp)`|将 Unix 时间戳转换为可读时间格式|`SELECT FROM_UNIXTIME(1747647900);` → `2025-05-19 17:45:00`|
|`DATEDIFF(date1, date2)`|返回两个日期相差的“天数”（date1 - date2）|`SELECT DATEDIFF('2023-10-15', '2023-10-10');` → `5`|
|`TIMEDIFF(time1, time2)`|返回两个时间之间的差值（格式是 HH:MM:SS）|`SELECT TIMEDIFF('10:30:00', '08:00:00');` → `02:30:00`|
|`DATE_ADD(date, INTERVAL n unit)`|在日期上加时间间隔|`SELECT DATE_ADD('2025-05-19', INTERVAL 5 DAY);` → `2025-05-24`|
|`DATE_SUB(date, INTERVAL n unit)`|在日期上减时间间隔|`SELECT DATE_SUB('2025-05-19', INTERVAL 7 DAY);` → `2025-05-12`|
|`YEAR(date)`|提取年份|`SELECT YEAR('2025-05-19');` → `2025`|
|`MONTH(date)`|提取月份|`SELECT MONTH('2025-05-19');` → `5`|
|`DAY(date)` 或 `DAYOFMONTH(date)`|提取日（几号）|`SELECT DAY('2025-05-19');` → `19`|
|`HOUR(datetime)`|提取小时数|`SELECT HOUR('2025-05-19 17:45:00');` → `17`|
|`MINUTE(datetime)`|提取分钟数|`SELECT MINUTE('2025-05-19 17:45:00');` → `45`|

---

需要我单独为你做一张“开发中常用时间函数速查表”PDF或 Markdown 表格版本吗？可以贴在项目文档里当工具书用 😎





==3.流程函数==

| 函数                                          | 功能                      |
| ------------------------------------------- | ----------------------- |
| IF(value, a, b)                             | 如果value为true，则返回a；否则返回b |
| IFNULL(a, b)                                | 如果a不为空，则返回a；否则返回b       |
| CASE WHEN [value] THEN [...] [ELSE ...] END | 也是条件语句                  |

==4.数字函数==

| 函数          | 功能                   | 举例                                                                                    |
| ----------- | -------------------- | ------------------------------------------------------------------------------------- |
| CEIL(x)     | 向上取整                 | SELECT CEIL(4.2) AS ceil_value; -- 结果：5 SELECT CEIL(-4.2) AS ceil_value; -- 结果：-4     |
| FLOOR(x)    | 向下取整                 | SELECT FLOOR(4.8) AS floor_value; -- 结果：4 SELECT FLOOR(-4.8) AS floor_value; -- 结果：-5 |
| MOD(x, y)   | 返回x / y的模            | SELECT MOD(10, 3) AS mod_value; -- 结果：1 SELECT MOD(10, 5) AS mod_value; -- 结果：0       |
| RAND()      | 返回0～1内的随机数           | SELECT RAND() AS random_value; -- 结果：例如：0.123456789                                   |
| ROUND(x, y) | 求取参数x的四舍五入值，保留小数点后y位 | SELECT id, ROUND(number, 2) AS rounded_value FROM numbers;                            |


==5.字符串函数==

|函数|功能|举例|
|---|---|---|
|CONCAT(s1, s2, ..., sn)|字符串连接，将s1, s2,…,sn几个字符串连接成一个字符串|SELECT LOWER('HeLLo WoRLd') AS result; -- 结果：'hello world'|
|LOWER(str)|返回字符串str的全小写字母|SELECT LOWER('HeLLo WoRLd') AS result; -- 结果：'hello world'|
|UPPER(str)|返回字符串str的全大写字母|SELECT UPPER('HeLLo WoRLd') AS result; -- 结果：'HELLO WORLD'|
|LTRIM(str)|返回字符串str去除左边空格的结果|SELECT LTRIM(' Hello World') AS result; -- 结果：'Hello World'|
|RTRIM(str)|返回字符串str去除右边空格的结果|SELECT RTRIM('Hello World ') AS result; -- 结果：'Hello World'|
|TRIM(str)|返回字符串str去除前导和尾随空格的结果|SELECT TRIM(' Hello World ') AS result; -- 结果：'Hello World'|
|SUBSTRING(str, pos, len)|返回字符串str从pos开始len个字符的字符串|SELECT SUBSTRING('Hello World', 7, 5) AS result; -- 结果：'World'|
|REPLACE(str, a, b)|返回字符串str，将其中的字符串a全部替换为字符串b|SELECT REPLACE('Hello World', 'World', 'MySQL') AS result; -- 结果：'Hello MySQL'|
|LPAD(str, len, pad)|返回字符串str从左边填充pad字符后，到len长度的结果|SELECT LPAD('Hello', 10, '-') AS result; -- 结果：'-----Hello'|
|RPAD(str, len, pad)|返回字符串str从右边填充pad字符后，到len长度的结果|SELECT RPAD('Hello', 10, '-') AS result; -- 结果：'Hello-----'|

---


### 6. 常见存储引擎

| **引擎**         | **特点**                                | **适合场景**                                     | 当前状况                                   |
| -------------- | ------------------------------------- | -------------------------------------------- | -------------------------------------- |
| **InnoDB（默认）** | 支持事务、行级锁、外键约束、ACID 特性，性能优良。           | 高并发、事务性应用、需要保证数据一致性、支持外键约束的场景（例如在线交易系统、金融应用） | MySQL 默认引擎，功能强大，适合大多数场景，强烈推荐使用 InnoDB。 |
| **MyISAM**     | 不支持事务和外键，支持表级锁，读性能高，写性能较差。            | 以读操作为主的场景，如数据仓库、日志分析、静态内容存储等                 | 不推荐使用                                  |
| **MEMORY**     | 数据存储在内存中，读写速度非常快，但数据会丢失。              | 临时表、缓存数据、临时存储数据的场景。适用于需要快速存取的临时数据处理（例如会话存储）  | 直接上手 Redis 它不香吗                        |
| **ARCHIVE**    | 高压缩、适合存储大量只读数据。支持 INSERT 但不支持 UPDATE。 | 存档数据、历史数据存储场景，特别是数据量大且访问频率较低的情况              | 适合存储历史数据，例如三个月前或三年前的数据，常用于数据归档。        |
| **BLACKHOLE**  | 不存储数据，所有插入操作都被丢弃，读取为空。                | 数据复制、测试用的空引擎，模拟写入操作而不存储数据                    | 临时测试场景，也不推荐使用                          |

---


### 7. SQL 脚本


SQL 文件（.sql 文件）通常用于存储数据库操作的 SQL 脚本，这些脚本包含各种数据库管理和操作命令。

我们可以手动编写 SQL 脚本，也可以使用工具如 Navicat，将已创建的表和数据库导出为 SQL 文件。通过在其他地方运行该 SQL 文件，我们可以轻松重建之前的表和数据库。SQL 文件不仅可以作为备份使用，还可以共享给他人。

==1.导出 SQL 文件==
![](source/_posts/笔记：MySQL%20基础/image-20250410104022257.png)

> [!NOTE] 注意事项
> 1. 这里将数据库导出为 SQL 文件，包含库、表和数据的创建。
> 2. 也可以只导出表，运行 SQL 文件时不会创建数据库，而是将在当前数据库中创建表和数据。


==2.运行 SQL 文件==
![](source/_posts/笔记：MySQL%20基础/image-20250410104203864.png)


==3.SQL 文件示例==
```
/*
 Navicat Premium Data Transfer

 Source Server         : 192.168.136.7
 Source Server Type    : MySQL
 Source Server Version : 80041 (8.0.41)
 Source Host           : 192.168.136.7:3306
 Source Schema         : test

 Target Server Type    : MySQL
 Target Server Version : 80041 (8.0.41)
 File Encoding         : 65001

 Date: 10/04/2025 10:42:42
*/

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for characters
-- ----------------------------
DROP TABLE IF EXISTS `characters`;
CREATE TABLE `characters`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `alias` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL,
  `title` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL,
  `affiliation` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL,
  `power_level` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL,
  `description` text CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL,
  `status` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 52 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of characters
-- ----------------------------
INSERT INTO `characters` VALUES (1, '姜子牙', '吕尚', '太师', '阐教', '天尊', '姜子牙是封神演义中的重要人物，辅佐周武王推翻商朝，封神榜上位列仙班', '已死');
```


==4.运行建议==
有人说 Navicat 运行速度较慢，我们可以考虑使用 SQLyog 进行运行。

---


### NULL 值处理


1. ==IS NULL==：
	1. 当列的值是 NULL,此运算符返回 true。
2. ==IS NOT NULL==：
	1. 当列的值不为 NULL, 运算符返回 true。
3. ==< = >==：
	1. 比较操作符（不同于 = 运算符），当比较的的两个值相等或者都为 NULL 时返回 true。
















# ------

不像redis，不强制读写分离，但推荐



### 1. MySQL 最佳实践

#### 1.1. 高可用实现

1. ==主从之间==
	1. MySQL 集群采用主从复制架构
	2. 主从复制：
	3. 故障转移
	4. 主控制从：
	5. 实时备灾：
	6. 读写分离：
	7. 数据备份：
2. ==主主之间==：
	1. 信息同步：
	2. 选举领导：
3. ==负载均衡==：
	1. 使用 K8S 进行部署
4. ==数据存储==：
	1. 存储位置：
		1. 首选本地存储，如果使用 K8s 担心主节点迁移，数据丢失，可选择以下方式：
		2. 公有云（块存储）
		3. 私有云
		4. 主节点固定，即使用 nodelector，将 Master 固定在那上面，方式 Master 迁移没数据了，从节点迁移无所调谓，反正又会同步（慢点？）
	2. 存储方法：
		1. MyCat 实现分片（分库分表）

---


#### 1.2. 架构图


#### 1.3. 流程图


#### 1.4. 安全规划


#### 1.5. 节点规划


#### 1.6. 数据存储

首选本地文件存储，如果是使用 K8S 搭建 MySQL 集群，









### 2. Docker 安装 MySQL

```
docker pull mysql:latest


docker run --name mysql-container -e MYSQL_ROOT_PASSWORD=wq666666 -d -p 3306:3306 mysql:latest


docker run --name mysql-container -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:latest
- `--name mysql-container`：指定容器的名称为 `mysql-container`。
- `-e MYSQL_ROOT_PASSWORD=my-secret-pw`：设置 MySQL 根用户的密码为 `my-secret-pw`。
- `-d`：让容器在后台运行。
- `mysql:latest`：指定要使用的 MySQL 镜像。

```








### 3. 集群搭建





![](source/_posts/笔记：MySQL%20基础/image-20250406182807697.png)




























### 4. 一主多从(keepavlied + mycat + MHA


==1.MySQL 集群==

| IP                  | 主机名              | 角色                      |
| ------------------- | ---------------- | ----------------------- |
| 192.168.136.8       | mysql-master<br> | MySQL 主库 + Keepalived 主 |
| 192.168.136.9<br>   | mysql-slave1     | MySQL 从库 + Keepalived 备 |
| 192.168.136.10<br>  | mysql-slave2     | MySQL 从库 + Keepalived 备 |
| 192.168.136.100<br> | mysql-vip        | 绑定在 MySQL 主库上的 VIP      |


==2.MyCat 集群==

| IP              | 主机名       | 角色                     |
| --------------- | --------- | ---------------------- |
| 192.168.136.12  | mycat1    | MyCat 主 + Keepalived 主 |
| 192.168.136.13  | mycat2    | MyCat 备 + Keepalived 备 |
| 192.168.136.101 | mycat-vip | 绑定在 MyCat 主上的 VIP      |


==3.MHA 集群==

| IP             | 主机名     | 角色                   |
| -------------- | ------- | -------------------- |
| 192.168.136.16 | mha1    | MHA 主 + Keepalived 主 |
| 192.168.136.17 | mha2    | MHA 备 + Keepalived 备 |
| 192.168.136.20 | mha-vip | 绑定在 MHA 主上的 VIP      |

1. MySQL 集群环境搭建
	1. 安装 Docker
	2. 安装 Keepalived
2. MyCAT 集群环境搭建
	1. 安装 MyCat
	2. 安装 Keepalived
3. MHA 集群环境搭建
	1. 安装 MHA
	2. 安装 Keepalived
4. MHA 与 MyCat 和 MySQL ssh互通，无密码




### 5. 非 K8S 环境

#### 5.1. 一主多从

##### 5.1.1. MySQL 集群节点环境准备

| IP    | 主机名    |
| ----- | ------ |
| 136.8 | mysql1 |



==1.修改主机名==
```
hostnamectl set-hostname <host-name>
```


==1.安装和配置 Docker==


==2.安装 Keepalived 和其相关依赖==
```
# 1. 安装 Keepalived 相关依赖
sudo apt install -y conntrack libseccomp2


# 2. 安装 Keepalived
sudo apt-get install -y keepalived
```


==3.创建数据持久化目录==
```
# 1. 模版
mkdir -p /data/mysql/<host-name>/conf /data/mysql/<host-name>/data


# 2. MySQL 1
mkdir -p /data/mysql/mysql1/conf /data/mysql/mysql1/data


# 3. MySQL 2
mkdir -p /data/mysql/mysql2/conf /data/mysql/mysql2/data


# 4. MySQL 3
mkdir -p /data/mysql/mysql3/conf /data/mysql/mysql3/data
```


==4.安装 MySQL==
```
docker run -d \                             # 以“后台模式”运行容器（detached mode）
  --name <host-name> \                      # 指定容器名称，建议替换 <host-name> 为具体名字，如 mysql-master
  -p 3306:3306 \                            # 将宿主机的 3306 端口映射到容器的 3306 端口（MySQL 默认端口）
  -e MYSQL_ROOT_PASSWORD=wq666666 \        # 设置 MySQL root 用户的初始密码
  -v /data/mysql/master/conf:/etc/mysql/conf.d \   # 挂载配置文件目录（宿主机 -> 容器），方便修改 MySQL 配置
  -v /data/mysql/master/data:/var/lib/mysql \      # 持久化数据目录，防止容器删除后数据丢失
  mysql:8.0 \                               # 使用 mysql 官方镜像，版本为 8.0
  --character-set-server=utf8mb4 \         # 设置 MySQL 服务器默认字符集为 utf8mb4（支持 Emoji 和多语言）
  --collation-server=utf8mb4_unicode_ci    # 设置排序规则为 utf8mb4_unicode_ci，适用于多语言环境


docker run -d \
  --name <host-name> \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=wq666666 \
  -v /data/<host-name>/master/conf:/etc/mysql/conf.d \
  -v /data/<host-name>/master/data:/var/lib/mysql \
  mysql:8.0 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci


docker run -d \
  --name mysql1 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=wq666666 \
  -v /data/mysql/mysql1/data:/var/lib/mysql \
  mysql:8.0 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci


docker run -d \
  --name mysql1 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=wq666666 \
  -v /data/mysql/mysql1/conf:/etc/mysql/conf.d \
  -v /data/mysql/mysql1/data:/var/lib/mysql \
  mysql:8.0 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci


docker run -d \
  --name mysql2 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=wq666666 \
  -v /data/mysql/mysql2/conf:/etc/mysql/conf.d \
  -v /data/mysql/mysql2/data:/var/lib/mysql \
  mysql:8.0 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci



docker run -d \
  --name mysql3 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=wq666666 \
  -v /data/mysql/mysql3/conf:/etc/mysql/conf.d \
  -v /data/mysql/mysql3/data:/var/lib/mysql \
  mysql:8.0 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci
```


---

##### 5.1.2. 部署 MySQL 集群

1. 配置 MySQL 主从

2. 在 MySQL1 上配置
1.配主库文件
```
1. 在 /data/mysql/mysql1/conf 路径下创建 my.cnf


2. 配置这个 cnf # /data/mysql/mysql1/conf/my.cnf

[mysqld]                  # 表示接下来的配置都是用于 mysqld 服务端的
server_id=1               # 定义该 MySQL 实例的唯一 ID，用于区分主从之间的服务器，不可重复
log-bin=mysql-bin         # 启用二进制日志（binlog），并设置文件名前缀为 mysql-bin
binlog_format=ROW         # 设置 binlog 的格式为 ROW（行模式）
gtid_mode=ON              # 启用 **GTID（全局事务 ID）复制**。
enforce_gtid_consistency=ON   # - 启用 **GTID 一致性检查**，防止执行不兼容 GTID 的语句。必须在启用 GTID 模式之前设置为 ON，否则会报错。


[mysqld]
server_id=1
log-bin=mysql-bin
binlog_format=ROW
gtid_mode=ON
enforce_gtid_consistency=ON



[mysqld]
server_id=1  # 主库的 server_id 一般设为 1，唯一
log_bin=mysql-bin  # 启用 binlog 记录，binlog 用于记录主库的所有写操作
binlog_format=ROW  # 设置为 ROW 格式，保证数据一致性
gtid_mode=ON  # 启用 GTID 复制，确保事务的一致性
enforce_gtid_consistency=ON  # 强制 GTID 一致性
log_slave_updates=ON  # 启用从库写 binlog，防止级联复制中丢失数据
read_only=0  # 主库是可写的
"""
1. server_id：
	1. 唯一标识一个 MySQL 实例，是 MySQL 复制系统中的“身份证”，如果两个节点 server_id 相同，复制就会失败
	2. 所有参与复制的实例（包括主库和所有从库）必须设置且唯一
	3. 通常主库设为 1，从库设为 2、3、4
2. log_bin：
	1. 开启 binlog（二进制日志）并指定文件前缀。
	2. mysql-bin 是日志文件前缀，最终文件会像 mysql-bin.000001 这样。
3. binlog_format：
	1. 用于设置 binlog 的记录格式，主要影响 DML 语句（如 INSERT、UPDATE、DELETE）的记录方式。可选值包括：
	2. ROW：
		1. 记录每一行数据修改前和修改后的值，主从复制时能最大程度地保证数据一致性
		2. 特别适用于包含触发器、函数、副作用操作的场景，主从一致性最强
		3. 缺点是 日志体积大（尤其批量更新时），且不记录原始 SQL，调试难度高
	3. STATEMENT：
		1. 只记录原始 SQL 语句，不记录执行结果或变更的具体值
		2. 优点是 体积小、写入快，效率高
		3. 缺点是依赖主从执行环境一致，若使用 NOW()、UUID() 等非确定性函数，可能导致主从数据不一致
		4. 比如 NOW() 是获取当前时间，你前后两次调用，虽然 SQL 语句一致，但是值不同。
	4. MIXED：
		1. MySQL 会根据 SQL 的内容自动选择 STATEMENT 或 ROW 模式。
		2. 通常优先使用 STATEMENT，但在检测到不安全语句（如含 UUID() 等非确定性函数）时自动切换为 ROW，兼顾效率与一致性
	5. 补充：
		1. 我们常见的约束、索引、视图等操作其实本质上都是 DDL，只是常常未单独列出。
		2. 不论使用哪种 binlog_format，DDL 和 DCL 操作都会以原始 SQL 语句事件的形式写入 binlog，binlog_format 的设置只影响 DML 语句的记录方式
4. gtid_mode：
	1. 是否启用 GTID（全局事务 ID）复制机制，ON 是开启
	2. GTID 给每一个事务一个全局唯一的 ID，主从间按 ID 进行复制，推荐必开
	3. 注意事项：
		1. 启用前数据库中不能有非 GTID 事务，要确保环境干净
5. enforce_gtid_consistency：
	1. 是否强制所有事务兼容 GTID，ON 是开启
6. log_slave_updates：
	1. 让 “从库” 在执行主库的事务时，也写自己的 binlog，ON 是开启
	2. 这个选项一般在作为从库时才需要开启，如果实例只是单纯的主库，不同步别人的数据，那就用不上。但如果是双主架构（彼此做 “从库”），或者将来要做故障切换，那就必须打开它了。
7. read_only：
	1. 是否只读，可选值：
	2. 0：读写
	3. 1：只读
	4. 注意事项：
		1. 我们在数据库层面保持 read_only 允许写入，而由 MyCat 统一管理读写分离策略，从而简化故障切换时的配置变更。
"""


4.重启
docker restart <container-id> / <container-name>

```


> [!NOTE] 注意事项：`binlog_format`：
> 1. `STATEMENT`（语句模式）：记录执行的 SQL 语句；
> 2. `MIXED`（混合模式）：智能切换；
> 3. `ROW`：记录实际变化的每一行，推荐用于主从复制。精确记录每一行的变更，适用于主从数据一致性要求较高的场景。
> 4. gtid？


==2.配从库文件==
```
1. 在 /data/mysql/mysql1/conf 路径下创建 my.cnf


2. 配置这个 cnf # /data/mysql/mysql1/conf/my.cnf

[mysqld]
server_id=2  # 从库2的唯一标识，主库为1，其他从库可以递增
relay_log=relay-bin  # 从库的中继日志文件前缀，用于存储主库的操作记录
read_only=1  # 设置为只读，防止从库接受写入操作
gtid_mode=ON  # 启用 GTID（全局事务ID）复制模式，保证复制一致性
enforce_gtid_consistency=ON  # 强制 GTID 一致性，确保只有符合 GTID 规则的事务被执行


[mysqld]
server_id=2
relay_log=relay-bin
gtid_mode=ON
enforce_gtid_consistency=ON



4.重启
docker restart <container-id> / <container-name>
```

> [!NOTE] 注意事项
> 1. 对对对，当 MHA 动态修改只读，和修改旧主为只读，改 log 是relay 还是mysql
> 2. 
> 3. 虽然mycat已经进行了读写分离，这里还用配置那么严格吗？（不严格，可以不用read_only=1配置只读）




==3.创建复制用户 repl 用户==
```
这是mysl命令啊，你要登录mysql客户端，或者用datagrip 和 navicat 来运行命令

-- 创建一个用于主从复制的用户 repl，允许任意主机连接
CREATE USER 'repl'@'%' 
  IDENTIFIED WITH mysql_native_password 
  BY 'wq666666';

repl：用户名
%：**允许从任意主机连接**（`%` 是通配符，匹配所有 IP 地址）
wq666666：用户密码




-- 授权该用户具有复制权限
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';


-- 刷新权限，使授权立即生效
FLUSH PRIVILEGES;

```


查看主库二进制位置（后面会用到）
```
SHOW MASTER STATUS;

```


![](source/_posts/笔记：MySQL%20基础/image-20250408141545381.png)
两个从节点可能相同，但是主节点不同，要注意了
![](source/_posts/笔记：MySQL%20基础/image-20250408141924184.png)

从库执行命令
```
这是mysql 命令啊


CHANGE MASTER TO 
  MASTER_HOST='主库 IP 地址',  -- 主库的 IP 地址，确保从库能够连接到主库
  MASTER_USER='repl',  -- 用于复制的用户，通常是具有 REPLICATION SLAVE 权限的用户
  MASTER_PASSWORD='Repl@2025',  -- 复制用户的密码，确保与主库上的复制用户一致
  MASTER_LOG_FILE='主库的 binlog 文件名',  -- 主库的当前 binlog 文件名，通过 `SHOW MASTER STATUS` 获取
  MASTER_LOG_POS=主库的 binlog 位置,  -- 主库的 binlog 位置，通过 `SHOW MASTER STATUS` 获取
  MASTER_AUTO_POSITION=1;  -- 启用 GTID 复制模式，自动管理复制位置


CHANGE MASTER TO 
  MASTER_HOST='主库 IP 地址', 
  MASTER_USER='repl', 
  MASTER_PASSWORD='Repl@2025',
  MASTER_LOG_FILE='主库的 binlog 文件名', 
  MASTER_LOG_POS=主库的 binlog 位置,
  MASTER_AUTO_POSITION=1;

最新版不用位置了，如果你使用 **GTID 复制**，只需要设置 `MASTER_AUTO_POSITION=1`，而不需要手动指定 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS`。
这是因为 `MASTER_AUTO_POSITION=1` 启用了 GTID（全局事务标识符）模式，GTID 模式会自动管理复制位置，而无需手动指定 `binlog` 文件和位置。

CHANGE MASTER TO 
  MASTER_HOST='192.168.136.8',
  MASTER_USER='repl',
  MASTER_PASSWORD='wq666666',
  MASTER_AUTO_POSITION=1;




启动从库复制状态
START SLAVE;


检查从库复制状态
SHOW SLAVE STATUS;


```


---











#### 5.2. 使用 Oracle MySql Operaotr 搭建 1 主 多从架构

==1.安装 MySQL Operator==

```
# 1. 部署 CRDs
kubectl apply -f https://raw.githubusercontent.com/mysql/mysql-operator/trunk/deploy/deploy-crds.yaml


# 2. 部署 MySQL Operator
kubectl apply -f https://raw.githubusercontent.com/mysql/mysql-operator/trunk/deploy/deploy-operator.yaml


# 3. 检查 Operaotr 部署状态
kubectl get deployment -n mysql-operator mysql-operator
```

> [!NOTE] 注意事项：
> 1. Oracle MySQL Operator 的 GitHub 地址：[Oracle MySQL Operator 地址](https://github.com/mysql/mysql-operator)，可以挑选版本或根据 README 进行安装
> 2. Oracle MySQL Operaotr 的安装脚本地址：[Oracle MySQL Operaotr 安装脚本地址](https://github.com/operator-framework/operator-lifecycle-manager/releases)

---


==2.为 MySQL root 用户创建 Opaque Secret==

```
# 1. 对密码进行 Base64 编码
echo -n "wq666666" | base64




# 2. 编写 Opaque Secret 资源 yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysql-root-secret
  namespace: mysql-operator
type: Opaque
data:
  rootPassword: xxxxxxx


# 3. 创建 Opaque Secret 资源
kubectl apply -f mysql-root-secret.yaml
```





3.创建 InnoDBCluster 资源

```
# mysql-primary.yaml 配置文件
apiVersion: mysql.oracle.com/v2            # 使用 MySQL Operator 的版本，指定使用 MySQL 2.x 版本
kind: InnoDBCluster                        # 类型为 InnoDBCluster，表示这是一个 MySQL InnoDB 集群
metadata:
  name: mysql-cluster                      # MySQL 集群的名称
  namespace: mysql-cluster   # 集群所在的命名空间，确保创建在正确的命名空间下
spec:
  secretName: mysql-root-secret  # 引用 Kubernetes 中存储 MySQL root 用户密码的 Secret
  instances: 3  # 集群中的实例数，1 主实例，2 从实例
  tlsUseSelfSigned: true  # 如果没有外部证书，则启用自签名证书（推荐用于开发和测试环境）
  podSpec:  # 为每个 Pod 定义资源和配置
    resources:
      requests:
        cpu: "1"  # 每个 MySQL 实例请求的 CPU 资源
        memory: "2Gi"  # 每个 MySQL 实例请求的内存资源
  volumeClaimTemplate:  # 定义存储卷的要求
    accessModes: ["ReadWriteOnce"]  # 存储访问模式，表示每次只能由一个 Pod 进行读写
    resources:
      requests:
        storage: 50Gi  # 每个实例请求的存储大小（50GB）




apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mysql-primary
  namespace: mysql-cluster
spec:
  secretName: mysql-root-secret
  instances: 3
  tlsUseSelfSigned: true
  podSpec:
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
  volumeClaimTemplate:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 50Gi
```









```
# mysql-primary.yaml

apiVersion: mysql.oracle.com/v2  # 使用 MySQL Operator 的 API 版本
kind: InnoDBCluster  # 资源类型为 InnoDBCluster，表示一个 MySQL 集群
metadata:
  name: mysql-primary  # 集群名称为 mysql-primary
  namespace: mysql-cluster  # 资源所在的 Kubernetes 命名空间
spec:
  secretName: mysql-root-secret  # 指定存储 MySQL root 密码的 Secret 名称
  instances: 3  # 设置集群中的实例数，这里有 1 个主节点和 2 个从节点
  tlsUseSelfSigned: true  # 如果没有外部证书，则启用自签名证书进行 TLS 加密通信
  podSpec:
    resources:
      requests:
        cpu: "1"  # 请求的 CPU 资源为 1 核
        memory: "2Gi"  # 请求的内存资源为 2Gi
  volumeClaimTemplate:
    accessModes: ["ReadWriteOnce"]  # 存储访问模式，表示每个节点只能有一个读写访问
    resources:
      requests:
        storage: 50Gi  # 每个 MySQL 实例请求 50Gi 的存储空间




# mysql-primary.yaml

apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mysql-primary
  namespace: mysql-cluster
spec:
  secretName: mysql-root-secret
  instances: 3
  tlsUseSelfSigned: true
  podSpec:
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
  volumeClaimTemplate:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 50Gi

```




























### 数据表设计

以下是为文学网站设计的数据库表结构，基于需求分析和实体-属性图，包含五个核心数据表：Users（用户表）、Categories（版块表）、Moderators（版主授权表）、Articles（文章表）和 Comments（留言表）。每张表按照用户查询中提供的格式，列出 **序号**、**数据项名称**、**数据类型**、**范围约束**、**关键数据项** 和 **含义**，以确保数据一致性、完整性和高效查询。

---

#### 1. Users（用户表）
**描述**：存储用户信息，用于身份验证和权限管理。

| 序号 | 数据项名称  | 数据类型      | 范围约束    | 关键数据项 | 含义                  |
|------|-------------|---------------|-------------|------------|-----------------------|
| 1    | UserID      | INT           | 无          | 是         | 用户表主键，自增      |
| 2    | Username    | VARCHAR(50)   | 非空，唯一  | 是         | 用户名，用于登录      |
| 3    | Password    | VARCHAR(255)  | 非空        | 否         | 用户密码（加密存储）  |
| 4    | Email       | VARCHAR(100)  | 非空，唯一  | 是         | 用户邮箱，用于联系    |
| 5    | Role        | ENUM          | 'Member', 'Moderator', 'Admin' | 否 | 用户角色              |
| 6    | CreatedAt   | TIMESTAMP     | 无          | 否         | 用户注册时间          |
| 7    | UpdatedAt   | TIMESTAMP     | 无          | 否         | 用户信息更新时间      |

---

#### 2. Categories（版块表）
**描述**：存储文章分类版块信息，如“茶余饭后”“风花雪月”。

| 序号 | 数据项名称   | 数据类型     | 范围约束   | 关键数据项 | 含义            |
|------|--------------|--------------|------------|------------|-----------------|
| 1    | CategoryID   | INT          | 无         | 是         | 版块表主键，自增|
| 2    | CategoryName | VARCHAR(50)  | 非空，唯一 | 是         | 版块名称        |
| 3    | Description  | TEXT         | 可为空     | 否         | 版块描述        |
| 4    | CreatedAt    | TIMESTAMP    | 无         | 否         | 版块创建时间    |

---

#### 3. Moderators（版主授权表）
**描述**：记录版主对特定版块的管理权限。

| 序号 | 数据项名称  | 数据类型   | 范围约束 | 关键数据项 | 含义                        |
|------|-------------|------------|----------|------------|-----------------------------|
| 1    | ModeratorID | INT        | 无       | 是         | 版主授权表主键，自增        |
| 2    | UserID      | INT        | 无       | 是         | 版主用户ID（外键，引用 Users.UserID） |
| 3    | CategoryID  | INT        | 无       | 是         | 管理的版块ID（外键，引用 Categories.CategoryID） |
| 4    | AssignedAt  | TIMESTAMP  | 无       | 否         | 授权时间                    |

---

#### 4. Articles（文章表）
**描述**：存储用户发表的文章信息，包括标题、内容和状态。

| 序号 | 数据项名称  | 数据类型     | 范围约束   | 关键数据项 | 含义                        |
|------|-------------|--------------|------------|------------|-----------------------------|
| 1    | ArticleID   | INT          | 无         | 是         | 文章表主键，自增            |
| 2    | UserID      | INT          | 无         | 是         | 作者用户ID（外键，引用 Users.UserID） |
| 3    | CategoryID  | INT          | 无         | 是         | 所属版块ID（外键，引用 Categories.CategoryID） |
| 4    | Title       | VARCHAR(200) | 非空       | 否         | 文章标题                    |
| 5    | Content     | TEXT         | 非空       | 否         | 文章内容                    |
| 6    | Status      | ENUM         | 'Draft', 'Pending', 'Published', 'Deleted' | 否 | 文章状态                    |
| 7    | CreatedAt   | TIMESTAMP    | 无         | 否         | 文章创建时间                |
| 8    | UpdatedAt   | TIMESTAMP    | 无         | 否         | 文章更新时间                |

---

#### 5. Comments（留言表）
**描述**：存储游客和会员对文章的留言。

| 序号 | 数据项名称  | 数据类型     | 范围约束 | 关键数据项 | 含义                        |
|------|-------------|--------------|----------|------------|-----------------------------|
| 1    | CommentID   | INT          | 无       | 是         | 留言表主键，自增            |
| 2    | ArticleID   | INT          | 无       | 是         | 关联文章ID（外键，引用 Articles.ArticleID） |
| 3    | UserID      | INT          | 可为空   | 否         | 留言用户ID（外键，引用 Users.UserID） |
| 4    | GuestEmail  | VARCHAR(100) | 可为空   | 否         | 游客邮箱                    |
| 5    | Content     | TEXT         | 非空     | 否         | 留言内容                    |
| 6    | CreatedAt   | TIMESTAMP    | 无       | 否         | 留言时间                    |

---

### 说明
- **关键数据项**：标记为“是”的字段是主键或外键，用于唯一标识记录或关联其他表。
- **范围约束**：
  - **非空**：该字段不能为空。
  - **唯一**：该字段值在表中必须唯一。
  - **可为空**：该字段可以为空，如 Comments 表中的 UserID 和 GuestEmail。
- **数据类型**：
  - **INT**：整数类型，用于ID字段。
  - **VARCHAR(n)**：变长字符串，n 为最大长度。
  - **TEXT**：长文本类型，用于文章和留言内容。
  - **ENUM**：枚举类型，限制取值范围，如 Role 和 Status。
  - **TIMESTAMP**：时间戳，记录创建和更新时间。
- **外键关系**：
  - Moderators.UserID 引用 Users.UserID
  - Moderators.CategoryID 引用 Categories.CategoryID
  - Articles.UserID 引用 Users.UserID
  - Articles.CategoryID 引用 Categories.CategoryID
  - Comments.ArticleID 引用 Articles.ArticleID
  - Comments.UserID 引用 Users.UserID（可为空）
- **其他约束**：Comments 表中 UserID 和 GuestEmail 至少一个不为空，以支持游客和会员留言。

---

### 总结
以上数据表设计满足文学网站的核心需求，包括用户管理、版块分类、版主授权、文章发布和留言功能。设计中充分考虑了字段类型、约束和外键关系，确保数据的一致性、完整性和查询效率。










### 1. 导图：[Map：数据类型和传参](Map：数据类型和传参.xmind)

---

 
### 2. ES 数据类型

| **名称**       | **范围**                                             | **描述**                                                                      | **赋值示例**                                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------ | -------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **数值类型**     |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| `byte`       | 1字节，-128 ~ 127                                     | 小范围整数                                                                       | `"age": 20`                                                                                                                                                                                                                                                                                                                                                                                                   |
| `short`      | 2 字节，-32768 ~ 32767                                | 稍大范围整数                                                                      | `"small_order_count": 300`                                                                                                                                                                                                                                                                                                                                                                                    |
| `integer`    | 4 字节，-2^31 ~ 2^31 -1                               | 常规整数                                                                        | `"sales_volume": 1234567890123`                                                                                                                                                                                                                                                                                                                                                                               |
| `long`       | 8 字节，-2^63 ~ 2^63 -1                               | 非常大的整数                                                                      | `"sales_volume": 1234567890123`                                                                                                                                                                                                                                                                                                                                                                               |
| `half_float` | 2 字节，3 ~ 5 位十进制精度                                  | 半精度浮点数牺牲精度，节省空间                                                             | `"sensor_data": 12.3`                                                                                                                                                                                                                                                                                                                                                                                         |
| `float`      | 4 字节，7 位十进制精度                                      | 常规浮点数，精度较低                                                                  | `"rating": 4.5`                                                                                                                                                                                                                                                                                                                                                                                               |
| `double`     | 8 字节，15 ~ 16 位十进制精度                                | 双精度浮点数，高精度，适合科学计算                                                           | `"measurement_value": 0.000000123`                                                                                                                                                                                                                                                                                                                                                                            |
|              |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| **字符串类型**    |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| `keyword`    |                                                    | 精确值                                                                         | `"status": "published"`                                                                                                                                                                                                                                                                                                                                                                                       |
| `text`       |                                                    | 可分词文本                                                                       | `"title": "Elasticsearch Guide"`                                                                                                                                                                                                                                                                                                                                                                              |
|              |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| **布尔类型**     |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| `boolean`    | 1字节，true、false                                     |                                                                             | `"is_online": true`                                                                                                                                                                                                                                                                                                                                                                                           |
|              |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| **日期与时间类型**  |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| `date`       | 支持从 1970-01-01T00:00:00Z 起的正负时间戳，实际范围取决于具体日期解析格式等  | 存储日期时间信息                                                                    | `"created_time": "2025-04-04T10:30:00Z"`（标准格式赋值）                                                                                                                                                                                                                                                                                                                                                              |
| `date_nanos` | 可存储到纳秒级别精度的日期时间，范围从 1970-01-01T00:00:00Z 起的正负纳秒时间戳 | 对时间精度要求极高的场景，如金融交易的精确时间戳、高精度实验数据的时间记录等                                      | `"trade_time": "2025-04-04T10:30:00.123456789Z"`（标准格式赋值）                                                                                                                                                                                                                                                                                                                                                      |
|              |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| **地理空间类型**   |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| `geo_point`  | 经度范围 -180.0 ~ 180.0，纬度范围 -90.0 ~ 90.0              | 存储由经度和纬度确定的地理坐标点                                                            | `"location": { 39.9042, 116.4074 }`<br>（39.9042 是纬度，116.4074 是经度）                                                                                                                                                                                                                                                                                                                                             |
| `geo_shape`  |                                                    | 存储由多个 `geo_point` 组成的复杂的地理形状数据，如点、线、面、多边形等                                  | **点**：`"area_shape": { "type": "point", "coordinates": [40.73, -73.93] }`<br>**矩形**：`"area_shape": { "type": "envelope","coordinates": [[-74, 40], [-73, 41]] }`<br>**圆形**：`"area_shape": { "type": "circle", "coordinates": [100.0, 0.0],   "radius": "100m“ }`<br>**任意形状**：`"area_shape": { "type": "polygon", "coordinates": [[ [100.0, 0.0], [101.0, 0.0], [101.0, 1.0], [100.0, 1.0], [100.0, 0.0] ]] }` |
|              |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| **复杂类型**     |                                                    |                                                                             |                                                                                                                                                                                                                                                                                                                                                                                                               |
| `object`     |                                                    | 用于在文档中嵌套其他文档，将相关字段组织在一起，如存储用户的详细信息（姓名、年龄、地址等在一个对象中）                         | `"user_info": { "name": "John", "age": 30, "address": "New York" }`                                                                                                                                                                                                                                                                                                                                           |
| `nested`     |                                                    | 与 `object` 类似，用于嵌套数组对象，使数组中的每个对象都能被独立地查询和聚合，如存储一篇博客的多条评论，每个评论又包含评论者信息、评论内容等 | `"comments": [ { "user": "Alice", "content": "Great post!" }, { "user": "Bob", "content": "Very helpful." } ]`                                                                                                                                                                                                                                                                                                |

---


### 3. MySQL 数据类型

| 名称           | 范围                                            | 描述                                             | 赋值示例                                                                  |
| ------------ | --------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------------- |
| **数值类型**     |                                               |                                                |                                                                       |
| `TINYINT`    | 1 字节，-128 ~ 127                               | 小范围整数                                          | `INSERT INTO table (tinyint_column) VALUES (20);`                     |
| `SMALLINT`   | 2 字节，-32768 ~ 32767                           | 稍大范围整数                                         | `INSERT INTO table (smallint_column) VALUES (300);`                   |
| `MEDIUMINT`  | 3 字节，-8,388,608 ~ 8,388,607                   | 中等范围整数                                         | `INSERT INTO table (mediumint_column) VALUES (500000);`               |
| `INT`        | 4 字节，-2^31 ~ 2^31 - 1（21 亿）                   | 常规整数                                           | `INSERT INTO table (int_column) VALUES (1000000);`                    |
| `BIGINT`     | 8 字节，-2^63 ~ 2^63 - 1                         | 非常大的整数                                         | `INSERT INTO table (bigint_column) VALUES (9223372036854775807);`     |
| `FLOAT`      | 4 字节，7 位十进制精度                                 | 常规浮点数，单精度，精度较低                                 | `INSERT INTO table (float_column) VALUES (4.5);`                      |
| `DOUBLE`     | 8 字节，15 ~ 16 位十进制精度                           | 双精度浮点数，精度高，适合科学计算                              | `INSERT INTO table (double_column) VALUES (0.000000123);`             |
| `DECIMAL`    | 精度可调，最大精度 65                                  | 精确小数，不丢失精度，适合金额等精度要求                           | `INSERT INTO table (decimal_column) VALUES (1234567890.12);`          |
|              |                                               |                                                |                                                                       |
| **字符串类型**    |                                               |                                                |                                                                       |
| `CHAR(n)`    | 0 ~ 255 字符                                    | 定长字符串，无论实际存储多少内容，始终占用 `n` 个字符的空间（不足部分用空格填充）。   | `INSERT INTO table (char_column) VALUES ('abc');`                     |
| `VARCHAR(n)` | 0 ~ 65535 字符                                  | 变长字符串，根据实际内容动态分配空间，`n` 表示可存储的最大字符数，不会浪费空间。     | `INSERT INTO table (varchar_column) VALUES ('Elasticsearch Guide');`  |
| `TINYTEXT`   | 最大 255 字符                                     | 变长字符串，用于存储短文本                                  | `INSERT INTO table (tinytext_column) VALUES ('Hello');`               |
| `TEXT`       | 最大 65,535 字符（64KB）                            | 变长字符串，用于存储较长文本                                 | `INSERT INTO table (text_column) VALUES ('This is a long text...');`  |
| `MEDIUMTEXT` | 最大 16,777,215 字符（16MB）                        | 变长字符串，用于存储更长文本                                 | `INSERT INTO table (mediumtext_column) VALUES ('Very long...');`      |
| `LONGTEXT`   | 最大 4,294,967,295 字符（4GB）                      | 变长字符串，用于存储极长文本                                 | `INSERT INTO table (longtext_column) VALUES ('Extremely long...');`   |
|              |                                               |                                                |                                                                       |
| **日期与时间类型**  |                                               |                                                |                                                                       |
| `DATE`       | 1000-01-01 ~ 9999-12-31                       | 仅日期                                            | `INSERT INTO table (date_column) VALUES ('2000-01-01');`              |
| `TIME`       | -838:59:59 ~ 838:59:59                        | 仅时间                                            | `INSERT INTO table (time_column) VALUES ('14:30:00');`                |
| `DATETIME`   | 1000-01-01 00:00:00 ~ 9999-12-31 23:59:59     | 日期 + 时间                                        | `INSERT INTO table (datetime_column) VALUES ('2025-04-04 10:30:00');` |
| `YEAR`       | 1901 ~ 2155                                   | 仅年份                                            | `INSERT INTO table (year_column) VALUES (2000);`                      |
| `TIMESTAMP`  | 1970-01-01 00:00:01 ~ 2038-01-19 03:14:07 UTC | 自动记录当前时间戳（非 Unix 中的时间戳，而是：YYYY-MM-DD HH:MM:SS） | `INSERT INTO table (timestamp_column) VALUES (CURRENT_TIMESTAMP);`    |

---


### Redis 数据类型














