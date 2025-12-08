---
title: SQL案例
summary: SQL面试中会碰到的手撕代码
date: 2025-12-07
authors:
  - admin
tags:
  - SQL
  - 手撕代码
---
## T1 
`留存率`

表1（用户登录行为数据，每天可能登录多次）  
`user_behavior(date, user_id)`

表2（用户信息）  
`user_info(user_id, gender, age)`

**求：**

1. 某个时间范围内（如 `20250801 ~ 20250831`），每天登录的用户数，以及 **登录用户数最多的日期**。

```sql
-- 1-1 每天登录的去重用户数
SELECT
    date,
    COUNT(DISTINCT user_id) AS login_users
FROM user_behavior
WHERE date BETWEEN '2025-08-01' AND '2025-08-31'
GROUP BY date
ORDER BY date;

-- 1-2 登录用户数最多的日期（如有并列会返回多行）
SELECT
    date,
    login_users
FROM (
    SELECT
        date,
        COUNT(DISTINCT user_id) AS login_users,
        RANK() OVER (ORDER BY COUNT(DISTINCT user_id) DESC) AS rnk
    FROM user_behavior
    WHERE date BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP BY date
) t
WHERE rnk = 1;

```
2. 定义 **次日留存率** = 某天登录的用户中，第二天依然登录的用户比例。问：`20250831` 用户的次日留存率。
```sql
WITH day0 AS (
    SELECT DISTINCT user_id
    FROM user_behavior
    WHERE date = '2025-08-31'            -- 当天登录的用户
),
day1 AS (
    SELECT DISTINCT user_id
    FROM user_behavior
    WHERE date = DATE_ADD('2025-08-31', INTERVAL 1 DAY)  -- 次日登录的用户
)
SELECT
    COUNT(*) AS day0_users,  -- 8 月 31 日总用户数
    SUM(CASE WHEN d1.user_id IS NOT NULL THEN 1 ELSE 0 END) AS retained_users, -- 次日仍登录
    SUM(CASE WHEN d1.user_id IS NOT NULL THEN 1 ELSE 0 END) * 1.0
        / COUNT(*) AS retention_rate      -- 次日留存率
FROM day0 d0
LEFT JOIN day1 d1
    ON d0.user_id = d1.user_id;

```

3. 接第 2 问，我们好奇：男性和女性、30 岁以上和 30 岁以下，哪一块用户的留存比较高？设计查询验证之。
```sql
WITH day0 AS (
    SELECT DISTINCT user_id
    FROM user_behavior
    WHERE date = '2025-08-31'
),
day1 AS (
    SELECT DISTINCT user_id
    FROM user_behavior
    WHERE date = DATE_ADD('2025-08-31', INTERVAL 1 DAY)
),
base AS (
    SELECT
        d0.user_id,
        ui.gender,
        CASE WHEN ui.age >= 30 THEN '30_plus' ELSE 'under_30' END AS age_group,
        CASE WHEN d1.user_id IS NULL THEN 0 ELSE 1 END AS retained_flag
    FROM day0 d0
    JOIN user_info ui
        ON d0.user_id = ui.user_id
    LEFT JOIN day1 d1
        ON d0.user_id = d1.user_id
)
SELECT
    gender,
    age_group,
    COUNT(*) AS day0_users,                        -- 分组当天用户数
    SUM(retained_flag) AS retained_users,          -- 分组次日仍登录用户数
    SUM(retained_flag) * 1.0 / COUNT(*) AS retention_rate
FROM base
GROUP BY gender, age_group
ORDER BY gender, age_group;

```
## License


Copyright 2025-present [Ginyee-W](https://ginyee-w.github.io/).