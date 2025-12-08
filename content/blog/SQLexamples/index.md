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
## T2
`环比`

有一张表 `creator_daily` 记录创作者每天的视频播放情况：

- `dt` ：日期（DATE）
- `creator_id` ：创作者 id
- `play_cnt` ：该创作者当天发布的视频的总播放量  
  > 只在创作者**当天有发布视频**时才在表中产生一行记录。

请使用 SQL（可以使用窗口函数）统计：在某一时间区间内（例如 `2024-01-01` ~ `2024-01-31`），  
**“连续 3 天都有发视频且播放量每天相对前一天环比增长超过 15%” 的创作者，占该时间段内所有发过视频创作者的比例。**
```sql
WITH base AS (
    SELECT
        dt,
        creator_id,
        play_cnt,
        LEAD(dt, 1) OVER (PARTITION BY creator_id ORDER BY dt)  AS dt2,
        LEAD(dt, 2) OVER (PARTITION BY creator_id ORDER BY dt)  AS dt3,
        LEAD(play_cnt, 1) OVER (PARTITION BY creator_id ORDER BY dt) AS play2,
        LEAD(play_cnt, 2) OVER (PARTITION BY creator_id ORDER BY dt) AS play3
    FROM creator_daily
    WHERE dt BETWEEN '2024-01-01' AND '2024-01-31'
),

-- 满足 “连续 3 天发视频且播放量连续两天环比 > 15%” 的创作者
three_day_creator AS (
    SELECT DISTINCT creator_id
    FROM base
    WHERE dt2 = DATE_ADD(dt, INTERVAL 1 DAY)
      AND dt3 = DATE_ADD(dt, INTERVAL 2 DAY)
      AND play2 >= play_cnt * 1.15
      AND play3 >= play2 * 1.15
),

-- 时间区间内所有发过视频的创作者总数
active_creator AS (
    SELECT COUNT(DISTINCT creator_id) AS total_cnt
    FROM creator_daily
    WHERE dt BETWEEN '2024-01-01' AND '2024-01-31'
)

-- 要求的创作者占比
SELECT
    COUNT(*) * 1.0 / (SELECT total_cnt FROM active_creator) AS creator_ratio
FROM three_day_creator;
```
## T3
`滑动窗口函数`

统计 **2021 年国庆头 3 天**（2021-10-01～2021-10-03）**每类视频每天的：**

1. 近 7 天（包含当天）**总点赞量** `sum_like_cnt_7d`
2. 近 7 天（包含当天）**最大单天转发量** `max_retweet_cnt_7d`

> 要求：  
> - 结果按照 **视频类型 `tag` 降序**，**日期 `dt` 升序** 排序；  
> - 假设 2021 年数据量足够多，至少每个类别在国庆头 3 天及之前一周的每天都有播放记录。

```sql
WITH d AS (
    -- 先按 “日期 + 类别” 汇总当天的点赞数和转发数
    SELECT
        t2.tag,
        DATE(t1.start_time) AS dt,
        SUM(t1.if_like)    AS sum_like,   -- 当天总点赞数
        SUM(t1.if_retweet) AS sum_re      -- 当天总转发数
    FROM tb_user_video_log AS t1
    JOIN tb_video_info      AS t2
          ON t1.video_id = t2.video_id   -- 通过 video_id 关联
    WHERE YEAR(t1.start_time) = 2021     -- 只统计 2021 年
    GROUP BY t2.tag, DATE(t1.start_time)
),

temp AS (
    -- 在每个 tag 内，按日期使用窗口函数统计近 7 天的数据
    SELECT
        tag,
        dt,
        -- 近 7 天（含当天）总点赞量
        SUM(sum_like) OVER (
            PARTITION BY tag
            ORDER BY dt
            ROWS 6 PRECEDING -- 滑动窗口函数
        ) AS sum_like_cnt_7d,

        -- 近 7 天（含当天）单日转发量最大值
        MAX(sum_re)  OVER (
            PARTITION BY tag
            ORDER BY dt
            ROWS 6 PRECEDING
        ) AS max_retweet_cnt_7d
    FROM d
)

-- 只取 2021-10-01 ~ 2021-10-03（国庆头三天）的记录
SELECT DISTINCT
    tag,
    dt,
    sum_like_cnt_7d,
    max_retweet_cnt_7d
FROM temp
WHERE dt BETWEEN '2021-10-01' AND '2021-10-03'
ORDER BY tag DESC, dt;
```
## License


Copyright 2025-present [Ginyee-W](https://ginyee-w.github.io/).