# MySQL 使用规范

以下规范适用在线交易（OLTP）系统的数据库。数据仓库与分析系统也可以参考。

## 命名规范

- 表名、字段名、索引名使用小写字母、数字，采用下划线分割
- 表名、字段名不超过 32 个字符
- 存储实体数据的表，名称使用名词，单数
- 索引名称采用 `idx_` 前缀，之后顺序跟随索引的字段名，字段名直接以下划线分割
- 不使用保留字
- 存储实体表间多对多对应关系的表，名称建议采用 `noun_verb_noun` 这样的模式。例如：
  `member_like_property`、`property_has_tag`。

SQL 语句中，

- 保留字使用全大写
- 字符串使用单引号（`'`）

```sql
-- 正确
SELECT id, title FROM xiaoqu WHERE id = 1
SELECT id, title FROM xiaoqu WHERE areacode = '000100010001'

-- 错误
select ID, title from XiaoQu where id = 1
SELECT id, title FROM xiaoqu WHERE areacode = "000100010001"
```

## 表的设计

### MySQL 存储引擎使用 InnoDB

> 不用纠结，没有特殊原因的情况下，作为 OLTP 的 MySQL 使用 InnoDB 引擎。

### 字符集使用 UTF-8

> Charset 为 `utf8`；Collation 为 `utf8_general_ci`。

### 正确使用时间类型

> MySQL 应当正确设置 `time_zone`。
>
> - 精确到秒的时间采用 `TIMESTAMP`
> - 精确到日期使用 `DATE`
> - 一般不使用 `DATETIME` 类型
> - **不允许使用字符串类型存储时间**

### 字段定义为 NOT NULL

> 真的需要 `NULL` 值吗？如果不确定，就将字段设置为 `NOT NULL`。

### 字段设置 DEFAULT 值

> 设置为 `NOT NULL` 的字段，需要设置一个缺省值。

### 不使用浮点类型（FLOAT、DOUBLE）

> 没有充分的理由，不要使用浮点数。

> 例如金额可以用分为单位，然后采用 `INT`。如果依然要以元为单位，可以采用 `DECIMAL`。

### 字段个数不超过 32 个

> 一个表有很多很多字段，是坏设计的味道。请再认真考虑设计是否正确。

### 不直接存储图片、音频、视频等大容量内容

> 请使用分布式文件系统来存储图片、音频、视频等内容。数据库里只存储文件的位置。

### 使用 INT UNSIGNED 来存储 IPv4 地址

> 使用 `INET_ATON` 将 IP 地址的字符串形式转换成数字形式；使用 `INET_NTOA` 将 IP 地址数字形式转换成字符串形式，以便查看。
>
> 当要查询某段的 IP 时，请参考以下示例：
>
> ```sql
> SELECT user_id FROM user_ip
> WHERE ip > INET_ATON('192.168.0.0') AND ip < INET_ATON('192.168.255.255')
> ```
>
> 当程序使用自带的函数进行 IP 地址的字符串形式与数字形式之间的转换时，需要注意数字的存储类型至少应为 32 位的无符号整型（如 `uint32_t`)，并注意字节顺。


## 索引

### 使用数字主键

> 存储实体数据的表，其主键应该是数字类型。

### 不使用联合主键

> 存储实体数据的表，不使用联合主键。
>
> 存储实体表间多对多对应关系的表（仅有两个字段）允许例外。

### 不使用外键

> 所有的表不建立外键约束。

### 联合索引字段数不超过 5 个

> 一个联合索引的字段数太多，很可能是设计得不好，还很难符合命名的规范。

### 前缀索引长度不超过 8 个字符

> 对字符串类型的字段建立索引，采用前缀索引，且长度不超过 8 个字符。


## SQL 语句

### 禁止在查询条件中对字段进行数学运算、函数调用、隐式类型转换

> 这类查询语句在使用索引时将非常困难。
>
> ```sql
> -- 禁止
> SELECT id FROM property WHERE NOW() - update_time < 3600
> SELECT id FROM property WHERE update_time + 3600 > NOW()
>
> -- 改为
> SELECT id FROM property WHERE update_time > NOW() - 3600
> ```
>
> ```sql
> -- 禁止
> SELECT id FROM property WHERE CHAR_LENGTH(title) > 20
> ```

> ```sql
> -- 假设字段 property.status 的类型为 TINYINT
> -- 禁止
> SELECT id FROM property WHERE status = '1'
>
> -- 改为
> SELECT id FROM property WHERE status = 1
> ```

### 禁止隐式类型转换

> 不仅在查询条件中禁止隐示类型转换，`INSERT`，`UPDATE` 也不允许隐式类型转换。
>
> ```sql
> -- 假设字段 property.status 的类型为 TINYINT
> -- 禁止
> INSERT INTO property (..., status) VALUES (..., '1')
> UPDATE property SET status = '1' WHERE id = '43'
>
> -- 改为
> INSERT INTO property (..., status) VALUES (..., 1)
> UPDATE property SET status = 1 WHERE id = 43
> ```

### 禁止使用 % 前导查询

> 尽量不使用 `LIKE` 查询，不得不用的情况下也禁止使用 `%` 前导查询。
>
> ```sql
> -- 禁止
> SELECT id FROM property WHERE title LIKE '%最%'
> ```

### 不使用联表查询

> OLTP 不使用 `JOIN` 联合查询。

### 不使用子查询

> 没有特别好的理由，OLTP 不允许使用子查询。

### 不使用负向查询

> 负向查询是指，如果查询条件描述的是不要什么数据，其余的都要。例如 `!=`、`<>`、`NOT EXISTS`、`NOT IN` 以及 `NOT LIKE` 等就是负向查询，它们利用索引将会很辛苦。

### 一次查询的结果集不超过 100 行

> 必要时使用 `LIMIT 100`

### LIMIT m, n，其中 m 应当小于 500

> 使用 `SELECT ... LIMIT offset, row_count` 或者 `SELECT ... LIMIT row_count OFFSET offset` 时，当 offset 小于 500 时，允许使用。
>
> ```sql
> -- 允许
> SELECT ... FROM property WHERE broker_id=? ORDER BY update_time LIMIT 40, 20
> -- 不允许
> SELECT ... FROM property WHERE areacode=? ORDER BY update_time LIMIT 4000, 20
> ```
>
> 能够不使用 offset 的情况应当避免，如下面的例子（其中 id 是主键），
>
> ```sql
> -- 建议
> SELECT ... FROM property WHERE broker_id=? AND id>? ORDER BY id LIMIT 20
> -- 避免
> SELECT ... FROM property WHERE broker_id=? ORDER BY id LIMIT 40, 20
> ```

### 避免使用 COUNT() 函数

> 能不使用就不使用，尽量用其他方法来解决。
>
> 例如判断经纪人是否有房源，可以不使用 `COUNT()` 函数，
>
> ```
> -- 正确
> SELECT 1 FROM propertys WHERE broker_id=? LIMIT 1
>
> -- 错误
> SELECT COUNT(*) FROM propertys WHERE broker_id=?
> ```

### 一次 COUNT() 可能扫描的行数应当确保小于 500 行

> `COUNT()` 函数需要扫描所有的结果集之后才能得出结果。而结果集的大小需要业务知识来判断（`EXPLAIN` 方法只能来来检验某一个条件下的当前情况）。因此需要使用 `COUNT()` 查询的代码应当经过审阅。
>
> ```sql
> -- 允许。审阅。经纪人的房源数不允许超过 200 套
> SELECT COUNT(*) FROM property WHERE broker_id=?
>
> -- 不允许。一个区域板块下的房源数量不定，可能非常多
> SELECT COUNT(*) FROM property WHERE areacode=?
> ```
>
> 其他聚合函数，例如 `SUM()`、`AVG()`、`MAX()` 等，同样适用。

### 统一使用 COUNT(*) 而不是 COUNT(1)

> 当统计行数时，
>
> - 统一使用 `COUNT(*)` 而不是 `COUNT(1)`。
> - 不使用 `COUNT(PK)` 或 `COUNT(column)`，除非真的是想统计 Nullable 字段的行数。
