---
title: SQL笔记
summary: SQL基础语法与案例分析
date: 2025-12-07
authors:
  - admin
tags:
  - SQL
  - 手撕代码
---
## 有用小函数
### NOT IN
```sql
SELECT *
FROM customers
WHERE customers.id NOT IN (
    SELECT customerId FROM orders
);
```
**作用:** 找出所有没有下过订单的顾客记录。它通过比较 `customers.id` 是否不在 `orders`表中的 `customerId` 列表中来实现。
### LIMIT
限制返回行数
```sql
SELECT DISTINCT salary
FROM Employee
ORDER BY salary DESC
LIMIT n - 1, 1
```
**作用:** 跳过前 n-1 个最高工资，取接下来的第 1 个
### ROWS BETWEEN... AND...

窗口函数里用来定义“窗口范围” 的语法，意思是对“当前这一行”来说，我要用“前后哪些行”来算这个 `SUM / AVG / COUNT `等。

**起点 / 终点 可用的关键词**

在 `ROWS BETWEEN 起点 AND 终点` 里，起点和终点可以是：

* `UNBOUNDED PRECEDING`：从组内**最前面**那行开始
* `n PRECEDING`：从当前行**往前第 n 行**
* `CURRENT ROW`：当前行
* `n FOLLOWING`：从当前行**往后第 n 行**
* `UNBOUNDED FOLLOWING`：一直到组内**最后一行**

都表示的是**相对当前行**的范围。

```sql
SELECT dept, dt, amount,
SUM(amount) OVER (
    PARTITION BY dept
    ORDER BY dt
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS running_sum
FROM sales;
```
**作用:** 在每个`dept`内，按日期`dt`顺序，为每一行算出“从这个部门最早的销售到当前日期为止的累计销售额”。
### DISTINCT
**1. 去重**
```sql
SELECT DISTINCT dept_id
FROM employees;
```
**2. 多列去重**

返回不同的多列组合
```sql
SELECT DISTINCT dept_id, job_title
FROM employees;
```
**3. 结合COUNT**
```sql
SELECT emp_id, dept_id,
COUNT(DISTINCT job_title) OVER (PARTITION BY dept_id) AS unique_jobs
FROM employees;
```
在不合并行的前提下，给每个员工所在部门统计该部门里有多少种不同的职位
### UNION ALL
将两列上下拼在一起
```sql
SELECT name, homework FROM students_a
UNION ALL
SELECT name, class FROM students_b;
```
* `UNION ALL`不会去重，`UNION`会去重
## 窗口函数
### 排名函数
**1. ROW_NUMBER**

组内顺序编号 <u>(无并列)</u>
```sql
SELECT dept, dt, amount,
ROW_NUMBER() OVER (PARTITION BY dept ORDER BY dt) AS rn
FROM sales;
```
* 1,2,3,4,5

**2. RANK**

排名，<u>(有并列), 会跳名次</u>
```sql
SELECT dept, amount,
RANK() OVER (PARTITION BY dept ORDER BY amount DESC) AS rnk
FROM sales;
```
* 1,1,3,4,5 

**3. DENSE_RANK**

排名，<u>(有并列), 不会跳名次</u>
```sql
SELECT dept, amount,
DENSE_RANK() OVER (PARTITION BY dept ORDER BY amount DESC) AS drnk
FROM sales;
```
* 1,1,2,3,4

**4. PERCENT_RANK**

在集合中有多少比例的值低于当前值，返回一个介于 0.0 到 1.0之间的小数。
```sql
SELECT 
    StudentID, 
    Score,
    PERCENT_RANK() OVER (ORDER BY Score) AS PercentRank
FROM Scores;
```
* 0.0,0.25,0.25,0.75,1.0
### LAG
“往前看”上一行（或前几行）的值，而不合并行。

* 往后用`LEAD`

**1. 上一行的值**



```sql
SELECT recordDate,
       Temperature,
       LAG(Temperature) OVER (ORDER BY recordDate) AS prev_Temperature
FROM weather;
```
* 气温和前一天对比

效果（示意）：

| recordDate | Temperature | prev_Temperature |
| ---------- | ----------- | ---------------- |
| 1号         | 10          | NULL             |
| 2号         | 12          | 10               |
| 3号         | 9           | 12               |

* 第一行没有“前一行”，所以 `prev_Temperature` 是 `NULL`。


**2. 按组使用：每个部门 / 每个用户内部自己比**

`PARTITION BY` 可以让每个组单独使用 LAG。

```sql
SELECT emp_id,
       month,
       sales,
       LAG(sales) OVER (
           PARTITION BY emp_id
           ORDER BY month
       ) AS prev_sales
FROM sales_table;
```
* 每个员工月度销量环比

**3. 往前不止一行：offset 参数**

```sql
LAG(sales, 2) OVER (PARTITION BY emp_id ORDER BY month) AS sales_2_months_ago
```

* 取的是“前两行”的 `sales`
* 如果前两行不存在 → 默认 `NULL`

可以配合 default 值：

```sql
LAG(sales, 1, 0) OVER (ORDER BY month) AS prev_sales
```
* 如果没有上一行，就用 `0` 代替。
### COUNT
```sql
SELECT emp_id, dept_id,
       COUNT(*) OVER (PARTITION BY dept_id) AS dept_size
FROM employees;
```
## 执行顺序
**1. FROM/JOIN**

**2. WHERE**

**3. GROUP BY**

**4. HAVING**

**5. SELECT**

**6. ORDER BY**
## License


Copyright 2025-present [Ginyee-W](https://ginyee-w.github.io/).
