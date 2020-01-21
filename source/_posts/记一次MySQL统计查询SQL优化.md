---
title: 记一次MySQL统计查询SQL优化
date: 2019-09-10 19:59:47
categories:
 - MySQL
tags:
 - MySQL
 - SQL
 - 优化查询
---

{% codeblock lang:sql %}

SELECT
	COL_B,
	MAX( CASE COL_A WHEN 'a' THEN 1 ELSE 0 END ) AS 'a',
	MAX( CASE COL_A WHEN 'b' THEN 1 ELSE 0 END ) AS 'b',
	MAX( CASE COL_A WHEN 'c' THEN 1 ELSE 0 END ) AS 'c' 
FROM
	Table 
GROUP BY
	COL_B 
ORDER BY 
    COL_C DESC
LIMIT 10

{% endcodeblock %}

COL_C上建立了一个普通索引。`50w`数据这个SQL的执行时间大概为`3s`。



{% tabs 优化 %}

<!-- tab 第一次优化 -->

我将一条SQL拆成两条

{% codeblock 分页 lang:sql %}

SELECT
	COL_B 
FROM
	Table
GROUP BY
    COL_B
ORDER BY
	COL_C DESC
LIMIT 10

{% endcodeblock %}

{% codeblock 查询 lang:sql %}

SELECT
	COL_B,
	MAX( CASE COL_A WHEN 'a' THEN 1 ELSE 0 END ) AS 'a',
	MAX( CASE COL_A WHEN 'b' THEN 1 ELSE 0 END ) AS 'b',
	MAX( CASE COL_A WHEN 'c' THEN 1 ELSE 0 END ) AS 'c'
FROM
	Table 
WHERE
	COL_B IN (...) 
GROUP BY
	COL_B
ORDER BY 
    COL_C DESC

{% endcodeblock %}

这条查询时间缩短到`0.5s`。在COL_B上加上索引之后，缩短到`0.1s`。

在第一次优化之后，`查询`的速度我已经可以接受了，但是`分页`我还是觉得太慢了。

<!-- endtab -->

<!-- tab 第二次优化 -->

对`分页`SQL执行EXPLAIN，发现Extra列有一个`Using filesort`，我想去掉这个。决定使用索引覆盖，在原来的COL_C的索引改为COL_C,COL_B的多列索引。但是这样还是不行，因为GROUP BY 使用的是COL_B，所以无法用到COL_C,COL_B的多列索引，因为索引中COL_C在COL_B左边。

既然是GROUP BY这里引起的，那么我可以不用`GROUP BY`，改为用`DISTINCT`

{% codeblock 分页 lang:sql %}

SELECT
    DISTINCT
	COL_B 
FROM
	Table
ORDER BY
	COL_C DESC
LIMIT 10

{% endcodeblock %}

这么一改时间由`1.4s`缩短至`0.1s`。

原来一条SQL查询需要`3s`，优化之后变成了两个`0.1s`的SQL。

<!-- endtab -->

{% endtabs %}


