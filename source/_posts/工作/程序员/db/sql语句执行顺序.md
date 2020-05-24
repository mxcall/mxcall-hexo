---
title: sql语句执行顺序
date: 2020-05-25 03:59:18
toc: true
categories: [工作, 程序员, db]
tags: [sql]

---

# sql语句执行顺序
## 参考链接
https://juejin.im/post/5c1c5301e51d451cdc394d13
https://www.cnblogs.com/wang666/p/9887631.html
join后添加外部行?  到底有没有这个概念
 
## 个人总结
### 执行顺序
```
1. From
2. ON
3. JOIN
4. WHERE
5. GROUP BY
6. SELECT
7. HAVING
8. ORDER BY
9. LIMIT
```
### 语法顺序
 
```
SELECT DISTINCT <select_list>
FROM <left_table>
<join_type> JOIN <right_table>
ON <join_condition>
WHERE <where_condition>
GROUP BY <group_by_list>
HAVING <having_condition>
ORDER BY <order_by_condition>
LIMIT <limit_number>
```
 
 
 
### 其它细节
 
- where 后面不能用select a as b的别名b, 因为此时根本没有执行select;
 
- having 后面可以用别名, 此时已经select 了
 
- A left join  B on B.name='test' 后面跟的条件, 仅仅只针对右表,  ~~即使写了左表条件也没用~~(同理, right join on 只针对左表, inner join on 针对2个表, 所以inner的条件放在on 和 where 都是一样的效果); **执行行流程**:  先根据on条件过滤B表, 再以左表为基准笛卡尔积, 所以左表有, 右表没有(或被on过滤掉的), 都会变为NULL;
 
-  A left join  B on B.name='test' 后面跟的条件,写了左边条件是有用的, 参考文章https://www.cnblogs.com/wang666/p/9887631.html
但是左边条件的作用不是过滤行, 而是过滤笛卡尔积的范围, 即仅仅笛卡尔积指定条件的左表行+右表行, 其他左表行对应的右表内容标记为NULL
所以总结说来, on 后面的条件, 不是过滤行(where才是过滤行), 而是过滤参与笛卡尔积的范围(**左右哪些行参与笛卡尔积**), 没参与的就标记内容为NULL
- 两张表的数据量比较大，又需要连接查询时，应该使用 FROM table1 JOIN table2 ON xxx的语法，避免使用 FROM table1,table2 WHERE xxx 的语法，因为后者会在内存中先生成一张数据量比较大的笛卡尔积表，增加了内存的开销
 
## 别人总结
left join中关于where和on条件的几个知识点：
    1.多表left join是会生成一张临时表，并返回给用户
    2.where条件是针对最后生成的这张临时表进行过滤，过滤掉不符合where条件的记录，是真正的不符合就过滤掉。
    3.on条件是对left join的右表进行条件过滤，但依然返回左表的所有行，右表中没有的补为NULL
    4.on条件中如果有对左表的限制条件，无论条件真假，依然返回左表的所有行,但是会影响右表的匹配值。也就是说on中左表的限制条件只影响右表的匹配内容，不影响返回行数。
结论：
    1.where条件中对左表限制，不能放到on后面
    2.where条件中对右表限制，放到on后面，会有数据行数差异，比原来行数要多
 