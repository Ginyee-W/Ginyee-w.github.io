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








## count()

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







## 函数对索引的影响

在MySQL中，**函数并非一定会导致索引失效**，其影响取决于函数的使用场景（是否作用于索引列的查询条件）。索引是基于列的原始值构建的，若函数改变了索引列的原始值，数据库无法利用索引的排序信息，会触发全表扫描；反之则不影响索引生效。

**一、会导致索引失效的情况**

1. **对索引列直接使用函数（WHERE/ON条件中）**
   若在查询条件中对索引列应用函数（如`LOWER()`、`DATE_FORMAT()`），会破坏索引列的原始值，导致索引无法被使用。
   示例：

   ```sql
   SELECT * FROM users WHERE LOWER(username) = 'john'; -- username是索引列，但函数导致索引失效
   ```

2. **索引列参与计算表达式**
   若索引列参与运算（如`column + 1`），数据库无法直接匹配索引值，会放弃使用索引。
   示例：

   ```sql
   SELECT * FROM orders WHERE price + 10 > 100; -- price是索引列，运算导致索引失效
   ```

**二、不会导致索引失效的情况**

**函数在索引范围之外使用**
若函数仅用于`SELECT`列表、`ORDER BY`、`GROUP BY`等非查询条件的位置，索引列的原始值未被修改，索引仍可正常生效。
示例：

```sql
SELECT UPPER(username) FROM users WHERE username = 'john'; -- WHERE中使用原始索引列，索引生效
```

**三、优化建议**

1. **使用生成列（持久化计算列）**
   对需要频繁通过函数查询的索引列，创建预计算的生成列并建立索引，避免实时函数计算。
   示例：

   ```sql
   ALTER TABLE users ADD username_lower VARCHAR(255) 
   GENERATED ALWAYS AS (LOWER(username)) STORED,
   ADD INDEX idx_username_lower (username_lower);
   ```

2. **应用层预处理**
   在应用程序中提前处理函数逻辑（如统一字符串大小写），避免在数据库查询条件中使用函数。

3. **用EXPLAIN验证索引使用**
   通过`EXPLAIN`语句分析查询计划，确认索引是否被有效利用。





## 索引失效反而提升效率

在大多数情况下，索引在查询中是用于提高性能的，因为它们加速了数据查找。然而，有些特殊情况下，索引的使用会导致性能下降，索引失效反而可能提升查询效率。以下是一些例子：

1. **小表查询**：
   - 对于非常小的表，MySQL可能会选择全表扫描（忽略索引），因为全表扫描的开销可能比通过索引逐行查找还要低。在这种情况下，索引失效不会损害性能，反而简化了查询。

2. **读取大部分或所有行**：
   - 当一个查询返回表中很大比例的行（如30%或更多）时，使用索引查找可能会耗时更多，因为数据库必须跳回主数据页以读取完整记录。全表扫描可能更有效，因为它可以逐行顺序读取数据。

3. **低选择性索引**：
   - 如果索引列的选择性非常低，例如一个布尔型字段，许多行有相同的值，那么依赖索引可能会产生不必要的开销。全表扫描可以避免索引的搜索和回表开销。

4. **频繁更新的表**：
   - 对于包含大量更新操作的表，索引的维护成本可能相对较高。尤其是在频繁更新索引列时，通过避免使用或减少复杂的索引可以减轻写操作的负担。

5. **复杂查询的优化选择**：
   - 对于复杂的多表联接查询，优化器有时可以选择执行计划中不使用某个索引（或部分失效）以提高整体联接和计算效率。

6. **数据分布与优化器误判**：
   - 在某些特定情况下，如果索引导致MySQL错误地估计数据分布或行数，手动禁用索引或提示优化器使用不同策略可能提升性能。











## License

Copyright 2025-present [Ginyee-W](https://ginyee-w.github.io/).

