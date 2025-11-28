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



## 索引
```sql
CREATE INDEX idx_name ON users(name); -- 为 'users' 表中的 'name' 列创建普通索引
ALTER TABLE users ADD PRIMARY KEY (id); -- 为 'users' 表的主键 'id' 添加主键索引
CREATE UNIQUE INDEX idx_email ON users(email); -- 为 'users' 表的 'email' 列创建唯一索引
```
### 1. B-Tree 索引（B树索引）
**特点**：B-Tree是最常见的索引类型，几乎所有的存储引擎都支持这种索引。它基于平衡二叉树的数据结构，能够高效地进行范围查询和精确查找。
**适用场景**：适用于大多数的查询需求，特别是需要进行范围搜索或有序数据的检索时效果更佳。例如，WHERE子句中的等值查询、ORDER BY、GROUP BY等操作。
### 2. Hash 索引
**特点**：基于哈希表实现，只有精确匹配索引列的查询才能使用这种索引。对于非等值查询（如范围查询），Hash索引是不适用的。
**适用场景**：适用于需要快速查找某个特定值的场景，例如主键、唯一键等。由于哈希表是基于数组的，所以检索速度非常快，但不适用于排序和范围搜索。
### 3. 组合索引（复合索引）
**特点**：由多个列组成的索引，可以覆盖一个或多个列上的查询。MySQL会自动优化组合索引的使用方式。
**适用场景**：当经常需要同时根据多个列进行查询时，使用组合索引可以显著提高查询效率。例如，对于WHERE子句中包含多个条件的查询，可以通过创建复合索引来加速。
### 索引失效反而提升效率

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

7. **SELECT 语句中对列进行运算**：
    - MySQL在进行索引选择时，会考虑查询条件中的表达式。如果查询条件中有函数或表达式（例如 `WHERE LENGTH(column) = 5`），MySQL可能会放弃使用索引。
   ```sql
   -- 不推荐
   SELECT * FROM table_name WHERE LENGTH(column) = 5;
   -- 推荐
   UPDATE column SET column = 'value' WHERE column = 'value';
   ```

8. **LIKE 查询以%开头**：
    - 当使用 `LIKE` 进行模糊查询时，如果搜索条件以通配符（如 `%`）开始，MySQL可能会放弃使用索引。
   ```sql
   -- 不推荐
   SELECT * FROM table_name WHERE column LIKE '%value';
   -- 推荐
   SELECT * FROM table_name WHERE column LIKE 'value%';
   ```

      对于需要匹配后缀的情况（即 `LIKE '%suffix'`），可以创建一个辅助列存储反转字符串，并基于此列进行前缀匹配。

      - **创建反向字符串**：

      ```sql
      ALTER TABLE users ADD reversed_username VARCHAR(255);
      UPDATE users SET reversed_username = REVERSE(username);
      CREATE INDEX idx_reversed_username ON users(reversed_username);

9. **数据类型不匹配**：
   - 如果查询条件中的列和索引列的数据类型不匹配，MySQL可能会放弃使用索引。例如，如果有一个字符串类型的索引列，但查询时将其作为数值类型使用。
   ```sql
   -- 不推荐
   SELECT * FROM table_name WHERE column = 123;
   -- 推荐
   UPDATE column SET column = 'value' WHERE column = 'value';
   ```




## License

Copyright 2025-present [Ginyee-W](https://ginyee-w.github.io/).

