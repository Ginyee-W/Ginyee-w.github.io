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
### 3.索引失效反而提升效率

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

10. **WHERE 子句中使用OR**：
   - 当 `WHERE` 子句中有多个条件，且这些条件是用 `OR` 连接的，MySQL可能会放弃使用索引。如果可能，使用复合索引来优化这种情况。
   ```sql
   -- 不推荐
   SELECT * FROM table_name WHERE column1 = 'value' OR column2 = 'value';
   -- 推荐
   ALTER TABLE table_name ADD INDEX idx_combined (column1, column2);
   ```

11. **ORDER BY 和 GROUP BY**：
    - 当使用 `ORDER BY` 或 `GROUP BY` 时，如果查询条件中的列不在索引中，MySQL可能会放弃使用索引。
   ```sql
   -- 不推荐
   SELECT * FROM table_name ORDER BY column;
   -- 推荐
   ALTER TABLE table_name ADD INDEX idx_column (column);
   ```

### 4.优化索引

1. **选择合适的索引列**：
   - **选择性高的列**：选择性高的列是指该列的值唯一的可能性很高，例如主键或唯一条目。这样的列适合作为索引。
   - **频繁查询的列**：经常在WHERE子句中出现的列应该被考虑建立索引。
   - **排序和分组依据的列**：用于ORDER BY、GROUP BY以及联接条件的列也应该被索引。

2. **创建复合索引**：
   - **复合索引**：复合索引是指包含两个或更多列的索引，其顺序也影响了性能。MySQL能够使用复合索引来优化多列查询。
   - **最左前缀原则**：在复合索引中，索引列的顺序非常重要。MySQL可以利用复合索引的最左前缀来进行有效的搜索。例如，创建(col1, col2)的复合索引比单独的(col2, col1)更高效，因为MySQL能够使用col1上的索引来快速定位数据。 

3. **避免冗余索引**：
   - **识别并删除重复和冗余的索引**：如果两个独立索引的列完全相同，那么只有第一个索引会被使用。可以使用以下查询来找出冗余索引：

4. **使用合适的索引类型**：
   - **主键和唯一索引**：强制数据的唯一性，加快查找速度。
   - **普通索引**：用于提高查询性能的列上创建。
   - **全文索引**：适用于全文搜索，如在TEXT或VARCHAR列上创建。

5. **定期分析和优化索引**：
   - **检查并重建索引**：使用`ANALYZE TABLE`和`OPTIMIZE TABLE`命令来确保索引统计信息是最新的，并可能通过删除并重新创建索引来优化它们。
   - **使用工具**：可以使用MySQL的自带工具如`mysqlcheck`或者第三方的优化工具如Percona Toolkit来进行索引维护。

6. **监控和调整**：
   - **监控查询性能**：使用慢查询日志（slow query log）来找出需要优化的查询，并进行相应的索引调整。
   - **分析执行计划**：使用`EXPLAIN`关键字来分析SQL语句的执行计划，查看是否正确使用了索引。

### 5.联合索引&最左匹配

1. **最左匹配原则（Leftmost Prefix）**
当使用联合索引进行查询时，只要在最左边连续的列上使用了相同的值，就可以使用这个索引。例如：
- 如果你创建了一个包含`column1`和`column2`的联合索引，那么查询条件中的组合可以是：
  - `WHERE column1 = 'value'`
  - `WHERE column1 = 'value' AND column2 = 'value'`
  但是，以下情况不能使用这个索引：
  - `WHERE column2 = 'value'`（因为最左边的`column1`没有被匹配）

2. **最左匹配原则的存在有几个原因：**
   1. **优化查询性能**：对于大多数查询，只需要使用索引的前缀就可以找到所需的数据。这样可以避免全表扫描，从而提高查询效率。
   2. **节约存储空间**：如果每种可能的组合都单独建一个索引，那么索引文件会变得非常大。最左匹配原则只对最左边连续的列进行索引，节省了存储空间。
   3. **一致性**：在最左匹配原则下，查询条件的顺序是固定的，这样数据库内部可以预先优化这些条件组合，从而提高执行效率。
3. **实际应用中的例子**
假设有一个表`my_table`，包含以下列：
- `id`（主键）
- `name`
- `age`
- `city`
你可以创建一个联合索引来优化查询：
```sql
CREATE INDEX idx_name_age ON my_table(name, age);
```
这个索引允许以下类型的查询使用索引进行快速查找：
- 高效查询：
  - `SELECT * FROM my_table WHERE name = 'John' AND age = 30;`
  - `SELECT * FROM my_table WHERE name = 'John';`
- 低效查询（无法使用索引）：
  - `SELECT * FROM my_table WHERE age = 30;`

### 6.B+树&哈希索引
**B+树索引**
   1. **范围查询**：B+树索引非常适合用于处理范围查询，比如`WHERE col2 > 10 AND col2 < 20`这样的条件。因为B+树的特性是按顺序存储数据，所以它可以很好地支持这种基于区间的查询。
   2. **排序和分组**：对于需要进行排序或分组的SQL语句，B+树索引可以提供快速的查找服务，尤其是当这些操作需要在索引上进行时。
   3. **多列组合索引**：在联合索引（复合索引）中，B+树索引可以高效地处理`WHERE col1 = 'value' AND col2 = 'value2'`这样的查询。
   4. **数据量大且频繁更新的场景**：对于数据量较大且更新操作不频繁的表，使用B+树索引是合适的选择。因为B+树在插入和删除时的维护成本相对较低。
   
**哈希索引**
   1. **等值查询**：由于哈希索引是通过直接计算键值的哈希码来定位数据的，因此对于精确匹配的等值查询（如`WHERE col = 'value'`）速度非常快。
   2. **集合成员检查**：当需要判断某个元素是否存在于一个集合中时，哈希索引是一个高效的选择。例如，`WHERE col IN ('value1', 'value2', 'value3')`这样的查询可以使用哈希索引来加速。
   3. **唯一性校验**：在某些场景下，比如主键或者唯一约束的检查，哈希索引可以快速验证数据的唯一性。
## License

Copyright 2025-present [Ginyee-W](https://ginyee-w.github.io/).

