---
title: SQLite学习笔记（二）
tags:
---

本篇博客继续学习SQLite相关语句
<!-- more -->
# SQLite 创建数据库
在任一目录下，sqlite3 创建命令的基本语法如下：
```shell
sqlite3 databaseName.db
```
另外也可以使用`.open`命令来建立新的数据库文件
首先进入 sqlite3 ,此时会默认有`sqlite> `开头，即进入了sqlite3 的命令行模式
```shell
sqlite3
```
然后输入 .open test.db ，如果 test.db 文件存在则会直接打开，否则就创建它
```sql
.open test.db
```
可以用 .database 查看是否创建成功，如：
```shell
sqlite> .database
main: /home/weiz3/learnpy/test.db r/w
```

可以在命令提示符中使用 SQLite .dump 点命令来导出完整的数据库在一个文本文件中，如：
```shell
sqlite3 test.db .dump >test.sql
```
.dump 命令会将转换整个 test.db 数据库的内容到 SQLite 的语句中，并转储到 ASCII 文本文件 test.sql中。可以用 test.sql 去恢复db，如：
```shell
sqlite3 test2.db < test.sql
```
此时会生成一个新的 test2.db 并且内容与 test.db一致

# SQLite 创建表
SQLite 的 **CREATE TABLE** 语句用于在任何给定的数据库创建一个新表。创建基本表需要涉及到命名表、定义列及每一列的数据类型
基本语法如下：
```shell
CREATE TABLE database_name.table_name(
   column1 datatype  PRIMARY KEY(one or more columns),
   column2 datatype,
   column3 datatype,
   .....
   columnN datatype,
);
```
如：
```sql
sqlite> CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);

```
其中 `sqlite>`表示当前在sqlite 的命令行模式中，该语句创建了一个 COMPANY 表， 其中 ID 作为主键， NOT NULL 的约束表示在表中创建记录时这些字段不能为 NULL

可以用 `.tables` 来验证是否创建表成功，该命令用于列出数据库中的所有表，如查看刚创建的表：
```sql
sqlite> .tables
COMPANY
```

可以使用 `.schema` 命令得到表的完整信息，如果 .schema 后不接表名，则会显示所有表的信息
```sql
sqlite> .schema COMPANY
CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);
```

# SQLite 删除表
SQLite 的 `DROP TABLE` 语句用来删除表定义及其所有相关数据、索引、触发器、约束和该表的权限规范。
> **使用此命令要谨慎，因为一旦一个表被删除，表中所有信息也将永远丢失**

基本语法如下，可以选择指定带有标明的数据库名称，如下所示：
```sql
DROP TABLE database_name.table_name;
```
删掉上面创建的 COMPANY 表：
```sql
sqlite> .tables
COMPANY
sqlite> DROP TABLE COMPANY;
sqlite> .tables
sqlite> 
```
可以看到开始有 COMPANY 表，在执行 DROP 后通过 .tables 没有任何表存在了

# SQLite Insert 语句
SQLite 的`INSERT INTO`语句作用是向数据库中的某个表中添加新的数据行
`INSERT INTO`语句有两种基本用法

```sql
INSERT INTO TABLE_NAME (column1, column2, column3,...columnN) 
VALUES (value1, value2, value3,...valueN);
```
第二种为：
```sql
INSERT INTO TABLE_NAME VALUES (value1,value2,value3,...valueN);
```
第一种语法中 column1, column2, ... ,columnN 是要插入数据的表中所对应列的名称，没有被声明为 NOT NULL字段的列会自动填上 NULL 值，被声明为 NOT NULL 约束的列必须要填上，否则会报错，如：
```sql
sqlite> INSERT INTO COMPANY (ID,  AGE, ADDRESS) VALUES (2 , 32,  'cali');
Runtime error: NOT NULL constraint failed: COMPANY.NAME (19)
```
第二种用法中 value1, value2, ... ,valueN 的顺序要与列在表中的顺序一致，并且数量要一致，当需要填 NULL 值时需要显式给出，否则也会报错
```sql
sqlite> INSERT INTO COMPANY  VALUES (2 , 'zheng' , 22,  'cali');
Parse error: table COMPANY has 5 columns but 4 values were supplied
```

# SQLite Select 语句
`SELECT`语句用于从 SQLite 数据库的表中获取数据，以结果表的形式返回数据，也被称为结果集。
SELECT 的基本语法如下：
```sql
SELECT column1, cloumn2,..., cloumnN FROM table_name;
```
其中 columnN 是表的列名，如果想获取所有可用的字段，可用使用如下语法：
```sql
SELECT * FROM table_name;
```
如打开一个已创建的表：
```sql
sqlite>.header on
sqlite>.mode column
sqlite> SELECT * FROM COMPANY;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
1   Paul   32   California  20000.0
2   Allen  25   Texas       15000.0
3   Teddy  23   Norway      20000.0
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0
6   Kim    22   South-Hall  45000.0
```
**其中前两个命令用来设置正确格式化的输出**
如果只想打开 COMPANY 表中指定的字段，则使用下面的查询：
```sql
sqlite> SELECT ID, NAME, SALARY FROM COMPANY;
ID  NAME   SALARY 
--  -----  -------
1   Paul   20000.0
2   Allen  15000.0
3   Teddy  20000.0
4   Mark   65000.0
5   David  85000.0
6   Kim    45000.0
```

列出所有在数据库中创建的表：
```sql
sqlite> SELECT tbl_name FROM sqlite_master WHERE type = 'table';
```
对于之前创建的表，会产生以下结果：
```sql
tbl_name
----------
COMPANY
```
列出所有表的完整信息：
```sql
sqlite> SELECT sql FROM sqlite_master WHERE type = 'table';
```
对于之前创建的表，会产生以下结果：
```sql
sqlite> SELECT sql FROM sqlite_master WHERE type = 'table';
sql                                
-----------------------------------
CREATE TABLE COMPANY(              
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),        
   SALARY         REAL             
)                                  
```

# SQLite 运算符
运算符是一个保留字或字符，主要用于 SQLite 语句的 WHERE 子句中执行操作，如比较和算术运算。运算符用于指定 SQLite 语句中的条件，并在语句中连接多个条件。
有下列四种运算符：
- 算术运算符
- 比较运算符
- 逻辑运算符
- 位运算符

## 算术运算符
| 运算符 | 描述                                    |
| ------ | --------------------------------------- |
| +      | 加法 - 把运算符两边的值相加             |
| -      | 减法 - 左操作数减去右操作数             |
| *      | 乘法 - 把运算符两边的值相乘             |
| /      | 除法 - 左操作数除以右操作数             |
| %      | 取模 - 左操作数除以右操作数后得到的余数 |

简单实例：
```sql
sqlite> .mode line
sqlite> select 10 + 20;
10 + 20 = 30
sqlite> select 10 - 20;
10 - 20 = -10
sqlite> select 10 * 20;
10 * 20 = 200
sqlite> select 10 / 5;
10 / 5 = 2
sqlite> select 12 %  5;
12 %  5 = 2
```

## 比较运算符
| 运算符   | 描述                                                       |
| -------- | ---------------------------------------------------------- |
| == ， =  | 检查两个操作数的值是否相等，如果相等则条件为真             |
| != ， <> | 检查两个操作数的值是否相等，如果不相等则条件为真           |
| >        | 检查左操作数的值是否大于右操作数的值，如果是则条件为真     |
| <        | 检查左操作数的值是否小于右操作数的值，如果是则条件为真     |
| >=       | 检查左操作数的值是否大于等于右操作数的值，如果是则条件为真 |
| <=       | 检查左操作数的值是否小于等于右操作数的值，如果是则条件为真 |
| !<       | 检查左操作数的值是否不小于右操作数的值，如果是则条件为真   |
| !>       | 检查左操作数的值是否不大于右操作数的值，如果是则条件为真   |

以之前创建的表为实例：
```sql
sqlite> SELECT * FROM COMPANY WHERE SALARY > 50000;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0
sqlite> SELECT * FROM COMPANY WHERE SALARY = 20000;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
1   Paul   32   California  20000.0
3   Teddy  23   Norway      20000.0
sqlite> SELECT * FROM COMPANY WHERE SALARY <> 20000;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
2   Allen  25   Texas       15000.0
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0
6   Kim    22   South-Hall  45000.0
sqlite> SELECT * FROM COMPANY WHERE SALARY >= 65000;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0
```

## 逻辑运算符
| 运算符  | 描述                                                                                                   |
| ------- | ------------------------------------------------------------------------------------------------------ |
| AND     | AND 运算符允许在一个 SQL 语句的 WHERE 子句中的多个条件的存在                                           |
| BETWEEN | BETWEEN 运算符用于在给定最小值和最大值范围内的一系列值中搜索值                                         |
| EXISTS  | EXISTS 运算符用于在满足一定条件的指定表中搜索行的存在                                                  |
| IN      | IN 运算符用于把某个值与一系列指定列表的值进行比较                                                      |
| NOT IN  | IN 运算符的对立面，用于把某个值与不在一系列指定列表的值进行比较                                        |
| LIKE    | LIKE 运算符用于把某个值与使用通配符运算符的相似值进行比较                                              |
| GLOB    | GLOB 运算符用于把某个值与使用通配符运算符的相似值进行比较。GLOB 与 LIKE 不同之处在于，它是大小写敏感的 |
| NOT     | NOT 运算符是所用的逻辑运算符的对立面。比如 NOT EXISTS、NOT BETWEEN、NOT IN，等等。它是否定运算符       |
| OR      | OR 运算符用于结合一个 SQL 语句的 WHERE 子句中的多个条件                                                |
| IS NULL | NULL 运算符用于把某个值与 NULL 值进行比较                                                              |
| IS      | IS 运算符与 = 相似                                                                                     |
| IS NOT  | IS NOT 运算符与 != 相似                                                                                |
| \|\|    | 连接两个不同的字符串，得到一个新的字符串                                                               |
| UNIQUE  | UNIQUE 运算符搜索指定表中的每一行，确保唯一性（无重复）                                                |

以之前创建的表为实例：
```sql
sqlite> SELECT * FROM COMPANY WHERE AGE >= 25 AND SALARY >= 65000;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0
sqlite> SELECT * FROM COMPANY WHERE AGE >= 25 OR SALARY >= 65000;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
1   Paul   32   California  20000.0
2   Allen  25   Texas       15000.0
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0
sqlite> SELECT * FROM COMPANY WHERE AGE IS NOT NULL;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
1   Paul   32   California  20000.0
2   Allen  25   Texas       15000.0
3   Teddy  23   Norway      20000.0
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0
6   Kim    22   South-Hall  45000.0
sqlite> SELECT * FROM COMPANY WHERE NAME LIKE 'Ki%';
ID  NAME  AGE  ADDRESS     SALARY 
--  ----  ---  ----------  -------
6   Kim   22   South-Hall  45000.0
sqlite> SELECT * FROM COMPANY WHERE NAME GLOB 'Ki*';
ID  NAME  AGE  ADDRESS     SALARY 
--  ----  ---  ----------  -------
6   Kim   22   South-Hall  45000.0
sqlite> SELECT * FROM COMPANY WHERE AGE IN ( 25, 27 );
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
2   Allen  25   Texas       15000.0
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0
sqlite> SELECT * FROM COMPANY WHERE AGE NOT IN ( 25, 27 );
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
1   Paul   32   California  20000.0
3   Teddy  23   Norway      20000.0
6   Kim    22   South-Hall  45000.0
sqlite> SELECT * FROM COMPANY WHERE AGE BETWEEN 25 AND 27;
ID  NAME   AGE  ADDRESS     SALARY 
--  -----  ---  ----------  -------
2   Allen  25   Texas       15000.0
4   Mark   25   Rich-Mond   65000.0
5   David  27   Texas       85000.0

```

## 位运算符
| 运算符 | 描述                                                             |
| ------ | ---------------------------------------------------------------- |
| &      | 二进制下的按位与运算                                             |
| \|     | 二进制下的按位或运算                                             |
| ~      | 二进制补码运算符是一元运算符，具有"翻转"位效应，即0变成1，1变成0 |
| <<     | 二进制左移运算符。左操作数的值向左移动右操作数指定的位数         |
| >>     | 二进制右移运算符。左操作数的值向右移动右操作数指定的位数         |
实例：
```sql
sqlite> .mode line
sqlite> select 60 | 13;
60 | 13 = 61
sqlite> select 60 & 13;
60 & 13 = 12
sqlite>  select  (~60);
(~60) = -61
sqlite>  select  (60 << 2);
(60 << 2) = 240
sqlite>  select  (60 >> 2);
(60 >> 2) = 15
```