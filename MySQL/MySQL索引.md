### 为什么要建立索引

> 多数情况下，不使用索引，试图通过其他途径来提高性能，纯粹是浪费时间（出自《MySQL技术内幕》）。

那索引是怎么提高性能的呢？![阅读更多…](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)

* 通过索引能获取数据的结束位置，从而跳过其他部分
* 定位算法，可以快速定位第一个匹配值

InnoDB总是使用**B树**来创建索引，对于这种索引，在使用<, <=,=,=>,!=,`BETWEEN`操作符时有会很有效率。

> Tip: BETWEEN在Django ORM对应`range`操作符。

```python
import datetime
start_date = datetime.date(2005, 1, 1)
end_date = datetime.date(2005, 3, 31)
Entry.objects.filter(pub_date__range=(start_date, end_date))
```

相当于

```sql
SELECT ... WHERE pub_date BETWEEN '2005-01-01' and '2005-03-31';
```

### 怎么建立索引

* 总结业务场景，分析出最常用的会在where中出现的字段

比如以我们的项目而言，`instance_name`，`user_id`， `check_date`出现的频率最高，所以这三个字段肯定需要建立索引。通过这样的索引，可以避免全表查找。

* 数据维度势

维度就是说表中容纳的非重复值的个数。我们尽量应该选择一些区分度高的，区分度＝count(distinct col)/count(*)，按照[美团](http://tech.meituan.com/mysql-index.html)的博客来讲，一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录。

* 不要滥用索引

由于在写入数据时，不仅要求写到数据行，还会影响所有的索引。所以索引建立越多，就会导致写入速度越慢。此外，索引会占据磁盘空间。

* 为字符串的前缀编索引

`短`索引可以减少索引空间，从而加快速度。

* 复合索引

比如地址，Province, City，通过这两个值得组合来建立索引。

* 最左前缀匹配

mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。

* 不要把索引列加入计算

尽量不要在where中的“=”的左边，进行计算。

### 怎么查询

1. 在where，order by语句中使用索引
2. 避免在where中去使用数据维度势低的，比如sex，isDeleted等
3. 如果是数字型字段，则使用数字类型
4. 尽量不要使用!=， like或者>, <，引擎可能会进行全表搜索，考虑使用between，union等来替代。

### 使用Explain来优化SQL

先大概解释下Explain返回的字段名吧。

| Column        | 意义        |      |
| ------------- | --------- | ---- |
| select_type   | select类型  |      |
| table         | 展示行的table |      |
| type          | join类型    |      |
| possible_keys | 索引的可能取值   |      |
| key           | 实际使用的索引   |      |
| key_len       | 使用索引的长度   |      |
| ref           | 跟索引       |      |
| rows          | 关键指标      |      |
| filtered      |           |      |
| Extra         |           |      |

具体可参考[MySQL Explain](http://dev.mysql.com/doc/refman/5.7/en/explain-output.html)。

截止目前，我们进行的都是单表查询，接下来看看多表的。

先来复习下left join，right join, inner join, outer join：

![SQL Joins](http://i.stack.imgur.com/VQ5XP.png)

Ref:

1. [美团点评团队](http://tech.meituan.com/mysql-index.html)
2. MySQL技术内幕

