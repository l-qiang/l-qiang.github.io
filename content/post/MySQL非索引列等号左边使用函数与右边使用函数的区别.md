---
title: "MySQL非索引列等号左边使用函数与右边使用函数的区别"
date: 2021-10-26T21:01:06+08:00
lastmod: 2021-10-26T21:01:06+08:00
categories: ["MySQL"]
tags: ["MySQL"]
keywords: ["MySQL", "非索引列", "函数", "等号"]
toc: false
draft: false
---

MySQL查询**upper(name) = 'A'** 与 **name = upper('A')**哪个更快，我想在`name`列为索引的情况下，都知道**后者**更快。那么`name`列为非索引的情况哪个更快呢？

<!--more-->

拥有多年代码经验的我，直觉就告诉我同样是**后者**更快，MySQL必定有优化，如果要说原因还确实触及到我的知识盲区了。

但是，MySQL可以告诉我原因。

```sql
EXPLAIN SELECT * FROM test.tt WHERE upper(`name`) = 'A';
SHOW WARNINGS;
EXPLAIN SELECT * FROM test.tt WHERE `name` = upper('A');
SHOW WARNINGS;
```

使用`EXPLAIN`后紧接着执行`SHOW WARNINGS`（也可使用`OPTIMIZER_TRACE`，详见[Chapter 8 Tracing the Optimizer](https://dev.mysql.com/doc/internals/en/optimizer-tracing.html)）。就可以看到MySQL优化后的SQL语句。

- upper(name) = 'A'

```sql
/* select#1 */ select `test`.`tt`.`id` AS `id`,`test`.`tt`.`name` AS `name` from `test`.`tt` where (upper(`test`.`tt`.`name`) = 'A')
```

- name = upper('A')

```sql
/* select#1 */ select `test`.`tt`.`id` AS `id`,`test`.`tt`.`name` AS `name` from `test`.`tt` where (`test`.`tt`.`name` = <cache>(upper('A')))
```

很明显，两者最大的区别就是`<cache>`标识了。

从MySQL的官方文档[Extended EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-extended.html)中可以看到如下说明。

> - <cache>(*expr*)
>
>   The expression (such as a scalar subquery) is executed once and the resulting value is saved in memory for later use. For results consisting of multiple values, a temporary table may be created and `<temporary table>` is shown instead.

`<cache>`标识的表达式只计算一次，并且计算结果会保存到内存以供之后使用。

