---
title: 数据库八股
summary: 笔面试中有关数据库的八股试题整理
date: 2025-11-20

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.


authors:
  - admin


tags:
  - 数据库
  - 八股

---



{{< toc mobile_only=true is_open=true >}}
## Like模糊查询优化

**1.合理使用索引**

- **前缀匹配**：使用 `LIKE 'prefix%'` 的形式，这种情况下 MySQL 能够利用索引，比如：

```sql
SELECT * FROM users WHERE username LIKE 'John%';
```

如果 `username` 字段有索引，前缀匹配会用到索引。



**2.使用反向索引**

对于需要匹配后缀的情况（即 `LIKE '%suffix'`），可以创建一个辅助列存储反转字符串，并基于此列进行前缀匹配。

- **创建反向字符串**：

```sql
ALTER TABLE users ADD reversed_username VARCHAR(255);
UPDATE users SET reversed_username = REVERSE(username);
CREATE INDEX idx_reversed_username ON users(reversed_username);

```

**3.限制扫描范围**

在 LIKE 查询中，如果可以通过其他条件进一步缩小搜索范围，请尽量加入这些条件。

```sql
SELECT * FROM users
WHERE created_at >= '2023-01-01'
  AND username LIKE 'John%';
```

**4.使用缓存**

如果相同的查询需要频繁执行，可以考虑在应用层实施缓存策略来减少数据库负载。

**5.使用专业工具**

对于非常大的数据集或者需要复杂文本处理和搜索功能，可以使用外部全文搜索引擎如 Elasticsearch、Solr 或者 Sphinx 来代替 MySQL 的 LIKE 操作。

通过这些方法优化 LIKE 查询，可以显著提升数据库的查询性能。应根据具体场景选择合适的优化策略。使用 EXPLAIN 分析查询执行计划，可以帮助确认优化效果。








## count(1)、count(*)与count(列名) 的区别?

在 SQL 中，`COUNT()` 函数用于统计行的数量，不同用法的核心差异体现在统计范围、逻辑与性能上，具体如下：

**1. COUNT(1)**

- **功能**：统计所有行的数量，不区分行内值的内容。
- **工作原理**：以常数 `1` 作为计数标识，对每一行均计数一次，逻辑与 `COUNT(*)` 一致。
- **效率**：多数数据库已对其优化，性能与 `COUNT(*)` 基本等效。

**2. COUNT(*)**

- **功能**：统计所有行的数量，包含所有列，不受列值是否为 `NULL` 的影响。
- **工作原理**：按行维度统计，不依赖列的具体数据。
- **效率**：是常用且性能较优的方式（如 InnoDB 引擎会直接统计数据页数目）。

**3. COUNT(column_name)**

- **功能**：仅统计指定列中值为**非 NULL** 的行数。
- **工作原理**：遍历列中每一行，仅计数非 `NULL` 的记录。
- **适用场景**：需过滤 `NULL` 值的场景（如统计特定字段有值的记录数）。

**性能对比**

- `COUNT(*)` 与 `COUNT(1)`：现代数据库已优化，性能基本一致。
- `COUNT(column_name)`：性能相对较低（需逐行检查列值是否为 `NULL`，尤其是列中 `NULL` 较多时）。

**用法选择**

- 优先选 `COUNT(*)`：语义清晰且通常经过数据库优化。
- 选 `COUNT(column_name)`：仅当需要统计指定列非 `NULL` 值的行数时使用。

**小结**

三者的选择核心依据是**是否需要过滤 `NULL` 值**：
- 仅统计总行数：用 `COUNT(*)` 或 `COUNT(1)`（推荐 `COUNT(*)`）。
- 统计指定列非 `NULL` 行数：用 `COUNT(column_name)`。












## License

Copyright 2025-present [Ginyee-W](https://ginyee-w.github.io/).

