---
title: SQLite学习笔记（一）
tags:
---

# 简介
SQLite是一款轻型的数据库，是遵守ACID的**关系型数据库管理系统**，它包含在一个相对小的C库中，实现了自给自足的、无服务器的、零配置的、事务性的 SQL 数据库引擎。就像其他数据库，SQLite 引擎不是一个独立的进程，可以按应用程序需求进行静态或动态连接。SQLite 直接访问其存储文件。

<!-- more -->
# SQLite 命令
与关系数据库进行交互的标准 SQLite 命令类似于 SQL。命令包括 CREATE、SELECT、INSERT、UPDATE、DELETE 和 DROP。这些命令基于它们的操作性质可分为以下几种：
## DDL-数据定义语言
| 命令   | 描述                                             |
| ------ | ------------------------------------------------ |
| CREATE | 创建一个新的表、一个表的视图或数据库中的其他对象 |
| ALTER  | 修改数据库中的某个已有的数据库对象，如一个表     |
| DROP   | 删除整个表、表的视图或者数据库中的其他对象       |

## DML-数据操作语言
| 命令   | 描述         |
| ------ | ------------ |
| INSERT | 创建一条记录 |
| UPDATE | 修改记录     |
| DELETE | 删除记录     |

## DQL - 数据查询语言
| 命令   | 描述                         |
| ------ | ---------------------------- |
| SELECT | 从一个或多个表中检索某些记录 |

# SQLite 数据类型
SQLite 数据类型是一个用来指定任何对象的数据类型的属性。 SQLite 中的每一列，每个变量和表达式都有相关的数据类型。我们可以在创建表的同时使用这些数据类型

## SQLite 存储类
每个存储在 SQLite 数据库中的值都具有以下存储类之一：
| 存储类  | 描述                                                                |
| ------- | ------------------------------------------------------------------- |
| NULL    | 值为一个NULL值                                                      |
| INTEGER | 值为一个带符号的整数，根据值的大小存储在1, 2, 3, 4, 6 或 8字节中    |
| REAL    | 值为一个浮点值，存储为8字节的IEEE浮点数字                           |
| TEXT    | 值为一个文本字符串，使用数据库编码（UTF-8, UTF-16BE或UTF-16LE）存储 |
| BLOB    | 值为一个blob数据，完全根据它的输入存储                              |


# SQLite 语法
## 注释
SQL 注释以两个连续的 "-" 字符开始，并扩展至下一个换行符或直到输入结束，以先到者为准。也可以使用 C 风格的注释，以 "/\*" 开始，并扩展至下一个 "*/" 字符对或直到输入结束，以先到者为准。SQLite的注释可以跨越多行。
## 大小写敏感性
SQLite 是不区分大小写的，但也有一些命令是大小写敏感的，比如 GLOB 和 glob 在 SQLite 的语句中有不同的含义

## SQLite 语句
所有的 SQLite 语句可以以任何关键字开始，如 SELECT, INSERT, UPDATE, DELETE, ALTER, DROP 等，所有的语句以分号 ";" 结束。

### SQLite ANALYZE 语句
```sql
ANALYZE;
or
ANALYZE database_name;
or
ANALYZE database_name.table_name;
```

### SQLite AND/OR 子句
```sql
SELECT column1, column2, ... ,columnN
FROM   table_name
WHERE  CONDITION_1 {AND|OR} CONDITION_ 2;
```

### SQLite ALTER TABLE 语句
```sql
ALTER TABLE table_name ADD COLUMN column_def...;
```
### SQLite ALTER TABLE Rename 语句
```sql
ALTER TABLE table_name RENAME TO new_table_name;
```

### SQLite ATTACH DATABASE 语句
```sql
ATTACH DATABASE 'DatabaseName' As 'Alias-Name';
```

### SQLite BEGIN TRANSACTION 语句
```sql
BEGIN;
or
BEGIN EXCLUSIVE TRANSACTION;
```
### SQLite BETWEEN 子句
```sql
SELECT column1, column2, ...columnN
FROM   table_name
WHERE  column_name BETWEEN val_1 AND val_2;
```

### SQLite COMMIT 语句
```sql
COMMIT;
```

### SQLite CREATE INDEX 语句
```sql
CREATE INDEX index_name
ON table_name ( column_name COLLATE NOCASE );
```

### SQLite CREATE UNIQUE INDEX 语句
```sql
CREATE UNIQUE INDEX index_name
ON table_name ( column1, column2,...columnN);
```

### SQLite CREATE TABLE 语句 语句
```sql
CREATE TABLE table_name(
   column1 datatype,
   column2 datatype,
   column3 datatype,
   .....
   columnN datatype,
   PRIMARY KEY( one or more columns )
);
```

### SQLite CREATE TRIGGER 语句
```sql
CREATE TRIGGER database_name.trigger_name 
BEFORE INSERT ON table_name FOR EACH ROW
BEGIN 
   stmt1; 
   stmt2;
   ....
END;
```

### SQLite CREATE VIEW 语句
```sql
CREATE VIEW database_name.view_name  AS
SELECT statement....;
```

### SQLite CREATE VIRTUAL TABLE 语句
```sql
CREATE VIRTUAL TABLE database_name.table_name USING weblog( access.log );
or
CREATE VIRTUAL TABLE database_name.table_name USING fts3( );
```

### SQLite COMMIT TRANSACTION 语句
```sql
COMMIT;
```

### SQLite COUNT 子句
```sql
SELECT COUNT(column_name)
FROM   table_name
WHERE  CONDITION;
```

### SQLite DELETE 语句
```sql
DELETE FROM table_name
WHERE  {CONDITION};
```

### SQLite DETACH DATABASE 语句
```sql
DETACH DATABASE 'Alias-Name';
```

### SQLite DISTINCT 子句
```sql
SELECT DISTINCT column1, column2....columnN
FROM   table_name;
```

### SQLite DROP INDEX 语句
```sql
DROP INDEX database_name.index_name;
```

### SQLite DROP TABLE 语句
```sql
DROP TABLE database_name.table_name;
```

### SQLite DROP VIEW 语句
```sql
DROP VIEW view_name;
```

### SQLite DROP TRIGGER 语句
```sql
DROP TRIGGER trigger_name
```

### SQLite EXISTS 子句
```sql
SELECT column1, column2....columnN
FROM   table_name
WHERE  column_name EXISTS (SELECT * FROM   table_name );
```

### SQLite EXPLAIN 语句
```sql
EXPLAIN INSERT statement...;
or 
EXPLAIN QUERY PLAN SELECT statement...;
```

### SQLite GLOB 子句
```sql
SELECT column1, column2....columnN
FROM   table_name
WHERE  column_name GLOB { PATTERN };
```

### SQLite GROUP BY 子句
```sql
SELECT SUM(column_name)
FROM   table_name
WHERE  CONDITION
GROUP BY column_name;
```

### SQLite HAVING 子句
```sql
SELECT SUM(column_name)
FROM   table_name
WHERE  CONDITION
GROUP BY column_name
HAVING (arithematic function condition);
```

### SQLite INSERT INTO 语句
```sql
INSERT INTO table_name( column1, column2....columnN)
VALUES ( value1, value2....valueN);
```

### SQLite IN 子句
```sql
SELECT column1, column2....columnN
FROM   table_name
WHERE  column_name IN (val-1, val-2,...val-N);
```

### SQLite Like 子句
```sql
SELECT column1, column2....columnN
FROM   table_name
WHERE  column_name LIKE { PATTERN };
```

### SQLite NOT IN 子句
```sql
SELECT column1, column2....columnN
FROM   table_name
WHERE  column_name NOT IN (val-1, val-2,...val-N);
```

### SQLite ORDER BY 子句
```sql
SELECT column1, column2....columnN
FROM   table_name
WHERE  CONDITION
ORDER BY column_name {ASC|DESC};
```

### SQLite PRAGMA 语句
```sql
PRAGMA pragma_name;

For example:

PRAGMA page_size;
PRAGMA cache_size = 1024;
PRAGMA table_info(table_name);
```

### SQLite RELEASE SAVEPOINT 语句
```sql
RELEASE savepoint_name;
```

### SQLite REINDEX 语句
```sql
REINDEX collation_name;
REINDEX database_name.index_name;
REINDEX database_name.table_name;
```

### SQLite ROLLBACK 语句
```sql
ROLLBACK;
or
ROLLBACK TO SAVEPOINT savepoint_name;
```

### SQLite SAVEPOINT 语句
```sql
SAVEPOINT savepoint_name;
```

### SQLite SELECT 语句
```sql
SELECT column1, column2....columnN
FROM   table_name;
```

### SQLite UPDATE 语句
```sql
UPDATE table_name
SET column1 = value1, column2 = value2....columnN=valueN
[ WHERE  CONDITION ];
```

### SQLite VACUUM 语句
```sql
VACUUM;
```

### SQLite WHERE 子句
```sql
SELECT column1, column2....columnN
FROM   table_name
WHERE  CONDITION;
```
