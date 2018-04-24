---
title: "Mysql"
date: 2018-04-23T19:52:04+08:00
draft: true
tags:
 - database
---
### 最佳实践

**每写下一个where或者order by条件时，思考一下是否有索引可以走？尽可能重用索引，减少索引的个数！！！**

#### 表设计
1. 每个表必须有主键
 + 主键采用无业务含义的id字段，避免业务数据修改时影响到引用该记录的其他表数据；
 + 采用数据库自身提供的自增ID(一般用int unsigned auto_increment)可以满足大部分场景的需求，而且可以有效减少表与索引占用空间；
 - 将来有分库分表需求的表主键可采用UUID(可以参考com.cw.util.GUID类，采用自己的编码将40位UUID字符串表示减少到26位)，避免表数据合并时产生主键冲突；
 - 如果将自增ID暴露在外可能导致安全风险，例如竞争对手通过每天订单ID的增量可以获知网站的业务规模，则该表宜采用UUID； 
2. 表字段类型选择要慎重，字段长度够用就好，用于过滤或者统计的字段通常要定义为not null
 - is null过滤条件用不到任何索引，应尽力避免这种语句，将相关字段定义为not null，用特殊值代表值不存在。例如评论表的userId外键字段可以用0表示用户不存在(只是逻辑上的外键，实际上考虑到性能因素并不在数据库上创建外键)；
 - 例如评论表的userId外键字段可以用0表示用户不存在(只是逻辑上的外键，实际上考虑到性能因素并不在数据库上创建外键)；
3. 索引要参照实际执行的SQL来设计，尽量用一个索引去满足多个SQL的查询
 - 索引不是越多越好，索引多了严重影响插入和修改性能。一个表的索引个数不应超过10个，高频更新的表索引个数更应该严格控制；
 - 经常过滤或者排序的字段应按照先where后order by的字段顺序建立索引；
 - 因为数据库执行引擎在选择索引的时候采用的是左匹配原则，所以如果A索引包含了(F1, F2, F3)字段，就没有必要再定义一个包含(F1, F2)字段的索引了。可以利用这个规则尽量重用已有的组合索引，减少索引的总数量；
 - 若表采用的是InnoDB存储引擎，则非主键索引默认在叶子节点存储了表的主键，因此如果定义了索引(F1)，相当于定义了一个复合索引(F1, PK)。但是查询分析器并不知道这个”潜规则“，有些可以走索引的场景却走了排序。因此，建议所有非主键索引都在末尾显式地加上主键，这不会带来额外的成本，但对查询分析器更加友好；
4. 定义索引的时候，选取的字段必须是选择性(或称离散度、区分度)较高的字段，即该字段的取值范围较大，不是像状态字段一样只有若干个值。在选择性很小的字段上建立索引查询时用不上反过来却影响了更新性能；
5. 如果where, order by, select相关字段都在索引中，而且SQL查询可使用该索引，查询性能是最高的.如果SQL查询只用到了索引，不涉及表数据，则explain时Extra会出现Using index。
6. 如果某个字段值可以确认唯一(NULL例外)，为其定义索引时应选择创建unique index，这有助于提升数据的完整性与查询性能；
7. 尽量做到单表查询，通过一定的冗余设计减少表关联；

#### sql编写
1. 禁止用字符串拼接SQL中的参数值，必须使用带参数的PreparedStatement执行SQL,例如读取产品表的SQL：

```
select itemNo from Product where id=? (iBatis写法：select itemNo from Product where id=#id#)
```
只是传入ID不同，但SQL文本一模一样。这样做有2个好处：

- 对于相同的SQL文本，数据库可以直接重用缓存中的SQL文本分析结果与执行计划，提高了SQL执行效率；
- 避免了黑客在传入值中注入危险代码，假如黑客给password参数赋值"' or 1=1 or '1'='"，那么如下SQL就可以跳过登录检查：

```
String password="' or 1=1 or '1'='";
String sql = "select 1 from User where name='" + name + "' and password='" + password + "'";
```
对于like条件，不仅要用?的方式传递参数值，在传递之前还要对字符串进行转义：将\替换成\\\\，将%替换成\%，将_替换成\\_。例如：
```
jdbcTemplate.query("select name from User where name like ?", esc(name) + "%");
```

2. 禁止在where或者order by的字段中使用SQL函数，这会使所有的索引失效,例如按照创建时间过滤产品的SQL：

```
select itemNo from Product where date_add(gmtCreated, 1 interval day)>now()
``` 
以上SQL在gmtCreated字段上执行了函数，那就会导致MySQL必须对所有记录的gmtCreated值进行函数运算后才知道它是否满足>now()的条件，无法通过索引结构快速查找。如果不是字段上的函数还是可以使用的，例如上述SQL可以改成：

```
select itemNo from Product where gmtCreated>date_sub(now(), 1 interval day)
```
因为不等式的右值在查询表数据之前就可以计算完成。

3. 禁止在过滤条件中使用全模糊Like匹配:like '%abc%'这样的全模糊过滤条件无法用到任何索引；like 'abc%'这样的前缀过滤还是可以用到索引的。
4. 深层次翻页写法的优化,以下SQL：

```
select id, name, price, description from Product order by gmtCreated limit 200000, 100;
```
的执行效率是非常低的，可以建一个联合索引(gmtCreated, id)，然后用以下2个SQL取代上述一个SQL，执行效率有几个数量级的提升：

```
select id from Product order by gmtCreated limit 200000, 100;
select id, name, price, description from Product where id in (?, ...); 
// 如果第二个SQL中不加order by gmtCreated，就要在应用代码中按照前一个SQL返回的id顺序排序，切记！
```
如果id是一个按用户创建时间自增的序列，那么有一种更加优化的方法：

```
select id, name, price, description from Product where id>? order by id limit 100;
```
每次记住上次查询到的最大一个id即可，系统只需扫描100行记录。 

5. 尽量避免子查询,同样的一个查询结果，用不用子查询可能性能差距几百上千倍：

```
# 秒级
select id from Favourite
where productId in
      (select id from Product where vsn>date_sub(now(), interval 1 day)
# 毫秒级
select f.id
from Favourite f, Product p
where p.vsn>date_sub(now(), interval 1 day) and p.id=f.productId
```

6. 利用数据库特性SQL,MySQL提供了很多标准SQL的扩展，例如：

```
select ... for update
insert into ... on duplicate key update ...
insert ignore into ...
replace into ...
```

考虑“增加积分”这样一个功能，如果不采用特性sql，则流程如下：

```
 select qty from Point where userId=#userId#
```

根据该用户的积分记录是否存在，应用程序决定后续采用update还是insert,这样做有很严重的数据正确性问题。在一个多线程并发的场景下，两个线程都执行了select：1）若发现积分记录不存在，则两个线程都插入了新的积分记录；(积分表应该在userId字段建立唯一索引避免此类数据问题)；2）若发现积分记录存在，则后面执行的那个线程覆盖了前一个线程的更新记录；

实际上这个功能用一个SQL就可以解决：在userId字段上建立唯一索引；

```
insert into Point (userId, qty) values (#userId#, #qty#) on duplicate key update qty=qty+(#qty#)
```
采用select .. for update也可以解决，不过这个解决方案需要在事务中完成：

```
set auto_commit=0
select qty from Point where userId=#userId# for update
insert into Point (userId, qty) values (#userId#, #qty#) 或 update Point set qty=qty+(#delta#) where userId=#userId# -- 根据积分记录是否存在commit
```

select ... for update相比insert into ... on duplicate update key ...各有优缺点：

- 当积分记录不存在时，select ... for update还是会导致第二个并发事务失败（违反了userId唯一约束），而insert into ... on duplicate update key ...都能更新成功；
- 当需要控制积分不能为负时，select ... for update后应用程序可以直接判断，为负时就不必做后续更新操作。insert into ... on duplicate update key ...则需要在事务中做如下处理：

```
set auto_commit=0
insert into Point (userId, qty) values (#userId#, #qty#) on duplicate key update qty=qty+(#qty#)
select qty from Point where userId=#userId#
commit 或 rollback --  根据当前qty判断
```
后者用了代价极大的事务回滚方式才达到和前者一样的结果，因此，如果积分为负是经常发生的事情，推荐用前者。

#### 业务逻辑与事务原则
1. 只查询必要的数据,对于那些随时间不断增加的表数据，不分页一次性查询出所有数据将带来灾难，你会不定期看到OutOfMemoryError；在select子句中明确指定返回列是一种好的做法；
2. 尽力避免在循环中查询数据库,合并查询条件，一次查询数据库得到最终数据是更好的做法；嵌套循环中查询数据库更是要杜绝；
3. 考虑在应用层缓存数据,常访问且总量大的数据（例如：产品、用户）考虑存储在Memcached；如果变动不频繁且总量不大，但经常需要批量访问的数据（例如：权限树、币别列表），可以考虑放在本地缓存，并且在数据发生变动时通过MQ告知各服务器刷新缓存。
4. 事务
 - 事务是代价极大的操作，能不用就不用，除非数据完整性是第一优先级保证，例如余额操作；
 - 事务中的处理过程应该尽量短，处理过程需要的数据可以在事务启动之前完成准备；
 - 事务中禁止远程调用，当外部系统不可用时会让这个事务挂起直至超时失败；
 - 当两个启用了事务的业务流程访问共同的多个资源时，尤其要注意避免死锁，访问资源的顺序要严格一致：假设P1与P2都需要在事务更新Ta, Tb表，P1更新了Ta接着更新Tb，而P2更新了Tb接着更新Ta，此时就出现了死锁。死锁与一般的性能问题不一样，它是代码错误，必须予以解决。
5. 异步化与最终一致性。一个事务仅完成最核心的业务逻辑，其它相关处理可以考虑异步化。例如：下单时让客户成功创建订单是最重要的，而返积分、发送邮件可以后续完成，失败也不要紧只要能够重试成功。可通过创建异步任务调度执行，也可通过持久化MQ给相关系统发送通知。异步任务应该对自己的业务状态进行记录与判断，避免重复执行。
6. 统计。是否可以通过一些状态标志实现增量统计？例如客户最近30天的订单金额。主从复制后，在从库上统计，将统计结果更新到主库；统计的条件个数与组合变化大，是否可以通过lucene实现？

#### 一个索引优化的例子

以下分析结果基于MySQL 5.0，在其它版本上可能得到不同的分析结果。

表结构：

```
create table Notice (
	id int unsigned auto_increment,
	email varchar(64) not null,
	status smallint not null,
	contact varchar(64) not null,
	gmtCreated datetime not null,
	primary key(id)
);
```
需求：按照email、status、contact过滤Notice，其中email与status条件必选，contact条件可选，查询结果按照gmtCreated排序。 SQL一般是这么写的：

```
select email, status, contact, gmtCreated from Notice where email='a' and status=1 and contact in ('1', '2', '3') order by gmtCreated;
select email, status, contact, gmtCreated from Notice where email='a' and status=1 order by gmtCreated;
```
先看SQL(1)，直觉上，我们感觉应该这么设计索引：

```
create index ix_notice_email on Notice(email, status, contact, gmtCreated, id); // 所有非主键索引末尾都加上PK
```
这个索引的字段顺序与SQL(1)中的where及order by顺序是完全左匹配的，应该OK的吧？

很不幸，当我们执行explain时看到MySQL在最终排序时还是走了filesort，而filesort是性能的噩梦：

```
explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 and contact in ('1', '2', '3') order by gmtCreated;
```

这是为什么呢？ 这是因为索引的顺序是这样的：字段1按照顺序排列、字段1相同的前提下字段2按照顺序排列……

如果SQL是这样就没有问题：

```
explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 and contact='1' order by gmtCreated;
注：MySQL会将contact in ('1')这种单值的IN优化为contact='1'
```

前面3个条件确定了gmtCreated的范围，在这个范围内的gmtCreated本来就是排好序的，所以不需要额外的filesort排序。

但是由于SQL(1)中contact in ('1', '2', '3’)条件的存在，分段取出的数据的gmtCreated已经不是排好序的了（当然，局部还是排好序的），因此需要放到排序缓冲区进行最终的排序。

同样的道理：只要SQL在索引的某个字段有范围查询（<,>,<=,>=,<>,IN,OR,LIKE等不是=的操作）条件，后续的索引字段就无法用于排序了。

再看SQL(2)：

```
explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 order by gmtCreated;
```

的分析结果也走了filesort：

不过SQL(2)走filesort原因与SQL(1)不同，它是因为ix_notice_email索引的第三个字段是contact而不是gmtCreated，没有左匹配，所以不能直接利用索引排序。

我们可以简单地调整一下索引设置优化查询性能，把索引中的contact与gmtCreated的顺序换一下：

```
create index ix_notice_email on Notice(email, status, gmtCreated, contact, id); // 所有非主键索引末尾都加上PK
```

现在执行一下：

```
explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 and contact in ('1', '2', '3') order by gmtCreated;
explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 order by gmtCreated;
```

filesort消失了：

需要说明的是，如果满足email='a' and status=1 and contact in ('1', '2', '3') 条件的记录数很少，出现filesort未必是不可接受的（只是放到缓冲区进行了一次排序而已）。

在选择索引的时候，有时MySQL和我们想象中的过程并不相同。例如：

```
create index ix_notice_email on Notice(email, status, contact, gmtCreated, id); // 所有非主键索引末尾都加上PK
create index ix_notice_email2 on Notice(email, status, gmtCreated, contact, id); // 所有非主键索引末尾都加上PK
```

结果对于：

```
explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 and contact in ('1', '2', '3'); // 无order by gmtCreated
```

MySQL选择了ix_notice_email2而不是ix_notice_email。而对于：

```
explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 order by gmtCreated;
```

恰恰选择了ix_notice_email而不是ix_notice_email2（因而出现了filesort）。

如果把创建索引的执行顺序反一下：

```
create index ix_notice_email2 on Notice(email, status, gmtCreated, contact, id); // 所有非主键索引末尾都加上PK
create index ix_notice_email on Notice(email, status, contact, gmtCreated, id); // 所有非主键索引末尾都加上PK
```

后面一个语句调整过来了：

```
 explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 order by gmtCreated;
```

走了正确的ix_notice_email2索引，但：

```
explain select email, status, contact, gmtCreated from Notice where email='a' and status=1 and contact in ('1', '2', '3');
```

还是走了错误的ix_notice_email2索引。

这个问题可能只能通过use index显式指定索引解决。

至于显式指定哪个索引，要根据实际情况做出判断，例如上述例子，如果满足email='a' and status=1 and contact in ('1', '2', '3')条件的记录很少，对于：

```
select email, status, contact, gmtCreated from Notice where email='a' and status=1 and contact in ('1', '2', '3') order by gmtCreated;
select email, status, contact, gmtCreated from Notice where email='a' and status=1 and contact in ('1', '2', '3'); // 无order by gmtCreated
```

来说，ix_notice_email可能是更好的索引。调整如下：

```
select email, status, contact, gmtCreated from Notice use index(ix_notice_email) where email='a' and status=1 and contact in ('1', '2', '3') order by gmtCreated;
select email, status, contact, gmtCreated from Notice use index(ix_notice_email) where email='a' and status=1 and contact in ('1', '2', '3'); // 无order by gmtCreated
```

如果contact不需要出现在select字句中，ix_notice_email2索引的最后一个字段contact也就没有必要存在了。

TODO: 这个问题很复杂，需要深入学习分析。





索引的使用。
