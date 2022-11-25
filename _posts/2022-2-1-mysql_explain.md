---
layout: post
title: Mysql Explain解析
date: 2022-2-1 15:06:13 +0800
categories: MySQL
tags: explain 
author: Rxsi
---

* content
{:toc}


表结构
```sql
CREATE TABLE `test_table` (
  `id` int NOT NULL,
  `name` varchar(10) NOT NULL,
  `age` int NOT NULL,
  `sex` tinyint NOT NULL,
  `height` int NOT NULL,
  PRIMARY KEY (`id`),
  KEY `name_age_sex` (`name`,`age`,`sex`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

CREATE TABLE `test_table2` (
  `id` int NOT NULL,
  `name` varchar(10) NOT NULL,
  `age` int NOT NULL,
  `sex` tinyint NOT NULL,
  PRIMARY KEY (`id`),
  KEY `name_age_sex` (`name`,`age`,`sex`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```
从表结构可知，表中有一个联合索引，由 name, age, sex 三个字段构成，有最左匹配的机制
<!--more-->
## Explain语句
```sql
mysql> explain select * from test_table where age  = 1;
+----+-------------+------------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
| id | select_type | table      | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+------------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | test_table | NULL       | index | name_age_sex  | name_age_sex | 47      | NULL |    1 |   100.00 | Using where; Using index |
+----+-------------+------------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```
可见一个explain语句会分析sql语句的执行情况，其中共有以下几个字段
### id
表示当前语句的查询顺序，一般有三种情况：

1：id全为1，则同属于同一层，顺序执行
```sql
mysql> explain select test_table.* from test_table, test_table2 where test_table.id = test_table2.id;
+----+-------------+-------------+------------+--------+---------------+--------------+---------+--------------------+------+----------+-------------+
| id | select_type | table       | partitions | type   | possible_keys | key          | key_len | ref                | rows | filtered | Extra       |
+----+-------------+-------------+------------+--------+---------------+--------------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE      | test_table  | NULL       | index  | PRIMARY       | name_age_sex | 47      | NULL               |    1 |   100.00 | Using index |
|  1 | SIMPLE      | test_table2 | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | test.test_table.id |    1 |   100.00 | Using index |
+----+-------------+-------------+------------+--------+---------------+--------------+---------+--------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```
2：id不相同，则id大者先执行，id较小者后执行
```sql
mysql> explain select test_table.* from test_table where id = (select id from test_table2 where id = (select test_table3.id from test_table3 where test_table3.name = ''));
+----+-------------+-------------+------------+------+---------------+--------------+---------+-------+------+----------+--------------------------------+
| id | select_type | table       | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra                          |
+----+-------------+-------------+------------+------+---------------+--------------+---------+-------+------+----------+--------------------------------+
|  1 | PRIMARY     | NULL        | NULL       | NULL | NULL          | NULL         | NULL    | NULL  | NULL |     NULL | no matching row in const table |
|  2 | SUBQUERY    | NULL        | NULL       | NULL | NULL          | NULL         | NULL    | NULL  | NULL |     NULL | no matching row in const table |
|  3 | SUBQUERY    | test_table3 | NULL       | ref  | name_age_sex  | name_age_sex | 42      | const |    1 |   100.00 | Using index                    |
+----+-------------+-------------+------------+------+---------------+--------------+---------+-------+------+----------+--------------------------------+
3 rows in set, 1 warning (0.00 sec)
```
3：id既有相同，又有不同，则id大者先执行，id较小者按顺序由上至下执行
```sql
(root@yayun-mysql-server) [test]>explain select t2.* from (select t3.id from t3 where t3.name='')s1, t2 where s1.id=t2.id;
+----+-------------+------------+--------+---------------+---------+---------+-------+------+--------------------------+
| id | select_type | table      | type   | possible_keys | key     | key_len | ref   | rows | Extra                    |
+----+-------------+------------+--------+---------------+---------+---------+-------+------+--------------------------+
|  1 | PRIMARY     | <derived2> | system | NULL          | NULL    | NULL    | NULL  |    1 |                          |
|  1 | PRIMARY     | t2         | const  | PRIMARY       | PRIMARY | 4       | const |    1 |                          |
|  2 | DERIVED     | t3         | ref    | name          | name    | 63      |       |    1 | Using where; Using index |
+----+-------------+------------+--------+---------------+---------+---------+-------+------+--------------------------+
3 rows in set (0.00 sec)
```
### select_type
表示查询中每个 select 子句的类型（**简单or复杂**）

- SIMPLE：查询中不包含子查询或者UNION
- PRIMARY：查询中若包含任何复杂的子部分，最外层查询则被标记为PRIMARY
- SUBQUERY：在SELECT或WHERE列表中包含了子查询，该子查询被标记为SUBQUERY
- DERIVED：在FROM列表中包含的子查询被标记为：DERIVED（衍生）用来表示包含在from子句中的子查询的select，mysql会递归执行并将结果放到一个临时表中。服务器内部称为"派生表"，因为该临时表是从子查询中派生出来的
- UNION：若第二个SELECT出现在UNION之后，则被标记为UNION；若UNION包含在FROM子句的子查询中，外层SELECT将被标记为：DERIVED

### type
表示MySQL在表中找到所需行的方式，又称为“访问类型”，常见类型有：（性能从上到下递增）

- ALL：性能最差，全表查询，一般是由于查询的条件未建立索引
```sql
mysql> explain select * from test_table where height = 10;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | test_table | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using where |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

- index：遍历索引树，意味着所有查询数据都可以直接从索引树获取（注意与eq_ref的区别）
```sql
mysql> explain select id from test_table;
+----+-------------+------------+------------+-------+---------------+--------------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+--------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | test_table | NULL       | index | NULL          | name_age_sex | 47      | NULL |    1 |   100.00 | Using index |
+----+-------------+------------+------------+-------+---------------+--------------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

- range：范围查询，常见的关键字有：between，< , > , in 等，这个也会利用到索引，但是只需要扫描一定范围，因此性能高于`index`

  此外，对索引进行`like`模糊查询时，如果查询形式为`select * from test_table where name like '林%'`，这种方式会直接扫描`name`索引（如果有建立索引的话）进行匹配，因为索引本身是按照字典序进行排列的，因此也是可以进行范围查询。而如果查询形式为`select * from test_table where name like '%林' 或者 like '%林%'`，这种则只能通过全表查询
  ```sql
  mysql> explain select * from test_table where id in (3, 4);
  +----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
  | id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
  +----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
  |  1 | SIMPLE      | test_table | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    2 |   100.00 | Using where |
  +----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
  1 row in set, 1 warning (0.00 sec)
  ```

- ref：使用非唯一索引查询，比如下面的搜索索引`name`是非唯一的，可能存在同时匹配多条记录的情况
```sql
mysql> explain select * from test_table where name = "林";
+----+-------------+------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | test_table | NULL       | ref  | name_age_sex  | name_age_sex | 42      | const |    1 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

- eq_ref：使用**主键、唯一索引**查询，且查询的**目标比较值不是常量**，且是查询一条数据（即 = xx）
```sql
mysql> explain select test_table.* from test_table, test_table2 where test_table.id = test_table2.id;
+----+-------------+-------------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
| id | select_type | table       | partitions | type   | possible_keys | key     | key_len | ref                | rows | filtered | Extra       |
+----+-------------+-------------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE      | test_table  | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL               |    1 |   100.00 | NULL        |
|  1 | SIMPLE      | test_table2 | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | test.test_table.id |    1 |   100.00 | Using index |
+----+-------------+-------------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
4
```

- const：当查询的目标最多只会存在一条数据，比如查询**主键、唯一键**等，且**目标比较值是一个常量**，那么查询得到的结果将会被优化器当成常量处理，因此性能高。
```sql
mysql> explain select * from test_table where id = 1;
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | test_table | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

- system：const类型的特例，当查询的表只会有一行的情况下（系统表），使用system
- NULL：MySQL在优化过程中分解语句，发现所要查询的数据已经有别的地方存储了，不需要访问索引树，比如获取索引的最大/小值。
```sql
mysql> explain select min(id) from test_table;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
1 row in set, 1 warning (0.05 sec)
```

### table
当前查询的表名
### partitions
如果查询基于分区表，将会显示访问的是哪个区
### possible_keys
可能用到的索引，当查询的字段的上有索引时即会列出，但不一定会使用到
### key
显示实际用到的索引，没有则为NULL
```sql
mysql> explain select * from test_table where id = 1;
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | test_table | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
### key_len
索引长度，比如primary key 是int类型，那么长度为4，如果是组合索引，如 name_age_sex，那么长度为
### ref
显示在key列索引中，表查找值所用到的列或常量，一般比较常见为const或字段名称。
### rows
估算结果集的大小
### filtered
返回结果集的行数占需要读到的行数的百分比，因此值越大，代表查询有效率更高
### Extra
额外信息，常见值有：

- Using index：代表使用了索引
- Using where：代表使用了where查询条件，但是所查列没有被索引覆盖
- Using where Using index：表示使用了where查询条件，且所查列有被索引覆盖，但是不符合最左匹配原则，常出现在联合索引
- NULL：表示查询的数据，需要使用“回表”来实现
- Using index condition：查询中含有索引，且是条件判断，如 id > 1
- Using filesort：对查询后的结果再次排序，一般出现在where a = xx order by b，且a是索引而b不是索引；如果有联合索引(a, b)那么这种情况会直接使用联合索引。
