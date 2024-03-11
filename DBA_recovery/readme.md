# 密态权限-故障恢复

## 前言

**自动提交**：默认情况下，PostgreSQL处于这种模式。这意味着每当执行一个SQL语句时，如果这个语句不是已经在一个明确的事务块中（即，没有被`BEGIN`...`COMMIT`或`BEGIN`...`ROLLBACK`包围），它就会自动被视为一个单独的事务并在执行后立即提交。——说白了我们每次select默认为一次事务块

## 实验

  <u>我们开启事务块，进行插入，不commit，强制退出——模拟断电</u>

1. john登录密态模式

```
gsql -p 5432 -d fruit_1 -r -U john -C
```

2. john执行查询语句

```sql
select * from fruit;
```

```
 fruit_id | fruit_name | quantity 
----------+------------+----------
        1 | Apple      |      100
        2 | Banana     |      150
        3 | Cherry     |      200
(3 rows)
```

3. john开启事务块，并插入，之后不commit，立马关闭终端，模拟断电

```sql
begin;
```

```sql
insert into fruit(fruit_name,quantity) values ('b1',100);
```

```sql
select * from fruit;
```

```
fruit_id | fruit_name | quantity 
----------+------------+----------
        1 | Apple      |      100
        2 | Banana     |      150
        3 | Cherry     |      200
        5 | b1         |      100
(4 rows)

```

  此时不commit，直接关闭终端

4. 之后重新连接，进行查看，密态数据库如下：

```
fruit_1=> select * from fruit;
 fruit_id | fruit_name | quantity 
----------+------------+----------
        1 | Apple      |      100
        2 | Banana     |      150
        3 | Cherry     |      200
(3 rows)
```

  这个本质就是`undo-redo`日志的机制

