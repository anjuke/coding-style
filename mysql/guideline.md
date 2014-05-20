# MySQL 使用规范

## 命名规范

- 表名、字段名、索引名使用小写字母、数字，采用下划线分割
- 表名、字段名不超过 32 个字符
- 存储实体数据的表，名称使用名词，单数
- 索引名称采用 `idx_` 前缀，之后顺序跟随索引的字段名，字段名直接以下划线分割
- 不使用保留字

- 存储实体表间多对多对应关系的表，名称建议采用 `noun_verb_noun` 这样的模式。例如：  
  `member_like_property`、`property_has_tag`、

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

### 使用 INT UNSIGNED 来存储IPv4 地址

> 使用 `INET_ATON` 或者程序自带的函数将 IP 地址转换成数字。
> 使用 `INET_NTOA` 或者程序自带的函数将数字转换成 IP 地址以便查看。
> 当要查询某段的 IP 时，请参考以下查询：

```

SELECT user_id FROM user_ip WHERE ip > INET_ATON('192.168.0.0') AND ip < INET_ATON('192.168.255.255')

```

## 索引

### 使用数字主键

> 存储实体数据的表，其主键应该是数字类型。

### 不使用联合主键

> 存储实体数据的表，不使用联合主键。

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

### 不使用联表查询

> OLTP 不使用 `JOIN` 联合查询。

### 避免使用子查询

> 没有特别好的理由，OLTP 不允许使用子查询。

### 避免使用负向查询

> 负向查询是指，如果查询条件描述的是不要什么数据，其余的都要。例如 `!=`、`<>`、`NOT EXISTS`、`NOT IN` 以及 `NOT LIKE` 等就是负向查询，它们利用索引将会很辛苦。

### 禁止使用 % 前导查询

> 尽量不使用 `LIKE` 查询，不得不用的情况下也禁止使用 `%` 前导查询。
>
> ```sql
> -- 禁止
> SELECT id FROM property WHERE title LIKE '%最%'
> ```
