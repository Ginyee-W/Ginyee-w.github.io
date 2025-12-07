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
### 排序编号
**1. ROW_NUMBER**

组内顺序编号 <u>(无并列)</u>
```sql
SELECT dept, dt, amount,
ROW_NUMBER() OVER (PARTITION BY dept ORDER BY dt) AS rn
FROM sales;
```
**2. RANK**

排名，<u>(有并列), 会跳名次</u>
```sql
SELECT dept, amount,
RANK() OVER (PARTITION BY dept ORDER BY amount DESC) AS rnk
FROM sales;
```
**3. DENSE_RANK**

排名，<u>(有并列), 不会跳名次</u>
```sql
SELECT dept, amount,
DENSE_RANK() OVER (PARTITION BY dept ORDER BY amount DESC) AS drnk
FROM sales;
```
## License


Copyright 2025-present [Ginyee-W](https://ginyee-w.github.io/).
