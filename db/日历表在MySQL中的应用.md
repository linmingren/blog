   根据日期来统计是非常常见的需求， 一般我们都是把时间字段通过DATE_FORMAT函数转成日期，或者月份，然后对转换后的字段进行GROUP BY来达到统计的效果。
通过GROUP BY来统计日期一个常见的问题是， 如果某一天没有记录，那么GROUP BY的结果集里也不会有这一天， 这肯定不是我们想要的结果。 如果是统计当天注册人数，
那么我们希望的结果是仍然显示这个日期，只是结果是0. 如果是统计累计注册人数， 那么我们希望的结果前一天的结果。为了简单起见， 我们构建下面这个user表。
然后完成2个简单的统计

* 每天新增的注册人数
* 每周新增的注册人数

```sql
CREATE TABLE user (
    id int NOT NULL AUTO_INCREMENT PRIMARY KEY,
   	user_name VARCHAR(16) NULL comment '用户名',
    signed_up_at DATETIME NOT NULL comment '注册时间'
);

INSERT INTO user (user_name, signed_up_at)
VALUES
('test1', '2020-01-01 00:00:00'),
('test2', '2020-01-01 00:00:00'),
('test3', '2020-01-02 00:00:00'),
('test4', '2020-01-02 00:00:00'),
('test5', '2020-01-04 00:00:00'),
('test6', '2020-01-12 00:00:00'),
('test7', '2020-01-12 00:00:00'),
('test8', '2020-01-18 00:00:00'),
('test9', '2020-01-19 00:00:00'),
('test10', '2020-01-19 00:00:00');
```
  对第一个统计，我们通过下面的group by语句看能不能达到我们想要的结果
```sql
select DATE_FORMAT(signed_up_at,'%Y-%m-%d') as day, 
             count(*) 
             from user 
             group by day;
```
  结果是这样的, 数据是对的，但是有一些日期，比如1月3号没有注册人数，我们期望的结果是显示这一天注册人数为0， 而不是没有记录。
```concept
    day      count(*)
    2020-01-01	2
    2020-01-02	2
    2020-01-04	1
    2020-01-12	2
    2020-01-18	1
    2020-01-19	2

```
   对第二个统计，我们通过下面的group by语句看能不能达到我们想要的结果
  ```sql
 select week(signed_up_at) as week, 
 count(*) 
 from user 
 group by week;
  ```
   结果是这样的， 问题和按天GROUP BY的一样，比如第二周没有人注册，那么我们期望的结果是第二周注册人数为0， 而不是没有记录。
  ```concept
      week      count(*)
        0	     5
        2	     3
        3	     2
  ```
   上面的统计可能还是比较简单的， 如果老板要我们统计工作日的注册人数， 还有周末的注册人数， 甚至统计假日的注册人数（除了周末还有法定假日， 像国庆这种假日还有调休），那
 通过GROUP BY来统计实在太难了。这个时候就是所谓的“日历表”大显身手的时候了。
 
   简单来说，日历表就是一个普通的表，这个表中每条记录对应一天， 有一个字段来表示具体的日期， 另外有一些额外的字段可以用来保存业务相关的内容，
  比如这一天是不是假期，是不是周末， 是当年的第几个季度，第几个月等等，凡是需要判断某一个天是什么特殊日期的内容，都可以记录在这一行中，把这个日历表
  和业务相关的表进行join查询就能比较容易地对业务进行统计。
  
   那日历表里面的记录要怎么构造出来呢？ 首先日期的范围是可以预先估计出来的，开始时间应该是你的业务上线的那一天， 结束时间可以设计成上线后的10年， 全部不会超过4千条记录。
  如果你想得到很准确的天数，可以用datediff函数， 比如上线时间是2020年1月1号，预计上线到2030年12月31， 那么下面的sql语句可以得到准确的天数，也就是4017天。
  ```sql
  SELECT datediff('2030-12-31','2020-01-01');
  ```
  
    接下来可以定义日历表的具体字段，注意这些只是最常见的字段， 业务线可以添加其他相关的字段
   ```sql
   CREATE TABLE calendar_table (
   	dt DATE NOT NULL PRIMARY KEY,
   	y SMALLINT NULL comment '年份',
   	q tinyint NULL comment '季度',
   	m tinyint NULL comment '月份',
   	d tinyint NULL comment '天',
   	dw tinyint NULL comment '每周的第几天',
   	monthName VARCHAR(9) NULL comment '比如 一月',
   	dayName VARCHAR(9) NULL comment '比如 礼拜一',
   	w tinyint NULL comment '第几周',
   	isWeekday SMALLINT(1) NULL comment '是否工作日',
   	isHoliday SMALLINT(1) NULL comment '是否假日',
   	holidayDescr VARCHAR(32) NULL comment '假日的描述， 比如元旦'
   );
   ```
   
   可以用Java或者其他其他编程语言来初始化这个表格，不过下面介绍的是纯SQL语言的方法。 在sql语言里，第一个难点是如何模拟一般语言中的for循环。比如要for循环N次，在sql里的模式是
   
   ```sql 
   select 记录 from 表 
   或者
   select 记录 from (子查询)
```
只要构造出N个记录的表或者返回N个记录的子查询即可。 如果是第一个方法， 那就先构造一个有10条记录的条， 然后对这个表join多次(每join一次就相当于多了10倍的记录)就能达到目的。

```sql
CREATE TABLE ints ( i tinyint );
 
INSERT INTO ints VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9);
```
接下来先初始化日历表的dt字段，其他字段的值可以根据这个字段算出来。我们在from后面一共join了4次，那么会生成1万条记录，但是我们只要3985条，所以在where语句里进行限制就行了。
```sql
INSERT INTO calendar_table (dt)
SELECT DATE('2020-01-01') + INTERVAL a.i*1000 + b.i*100 + c.i*10 + d.i DAY
FROM ints a JOIN ints b JOIN ints c JOIN ints d
WHERE (a.i*1000 + b.i*100 + c.i*10 + d.i) <= 4017
ORDER BY 1;
```

执行完毕后，日历表里就有了从2020年2月2号到2030年12月31号的所有日期。接下来更新其他字段，

```sql
UPDATE calendar_table
SET isWeekday = CASE WHEN dayofweek(dt) IN (1,7) THEN 0 ELSE 1 END,
	isHoliday = 0,
	y = YEAR(dt),
	q = quarter(dt),
	m = MONTH(dt),
	d = dayofmonth(dt),
	dw = dayofweek(dt),
	monthname = monthname(dt),
	dayname = dayname(dt),
	w = week(dt),
	holidayDescr = '';
```

最后更新假日字段， 因为每年的假日不是固定的， 所以这些SQL只能根据国际发布的通知来写具体的代码。 下面举个例子是元旦，这是个固定的日期，所以写出来比较简单。其他假期可以按照这个模式来处理。
```sql
UPDATE calendar_table SET isHoliday = 1, holidayDescr = 'New Year''s Day' WHERE m = 1 AND d = 1;
```
   
  有了这个日历表， 我们就可以完美结果本文开头的2个统计了。第一个统计可以这么写.
 ```sql
select c.dt as day, IFNULL(u.signup_number,0) from calendar_table c
left join
(select DATE_FORMAT(signed_up_at,'%Y-%m-%d') as day, 
             count(*) as signup_number
             from user 
             group by day) u on c.dt = u.day
order by c.dt limit 20;
 ```
 结果如下, 可以看到那些没有用户注册的日期， 对应的注册次数都是0。
 ```sql
   day    signup_number
 2020-01-01	2
 2020-01-02	2
 2020-01-03	0
 2020-01-04	1
 2020-01-05	0
 2020-01-06	0
 2020-01-07	0
 2020-01-08	0
 2020-01-09	0
 2020-01-10	0
 2020-01-11	0
 2020-01-12	2
 2020-01-13	0
 2020-01-14	0
 2020-01-15	0
 2020-01-16	0
 2020-01-17	0
 2020-01-18	1
 2020-01-19	2
 ```
 第二个统计可以这么写(需要把sql_mode=only_full_group_by禁用)
 ```sql
select c.w as week, IFNULL(u.signup_number,0) as signup_number from calendar_table c
left join
( select week(signed_up_at) as week, 
 count(*) as signup_number
 from user 
 group by week) u on c.w = u.week
group by week order by week
 ```
 结果如下，可以看到如果某一周内没有会员注册，那么注册人数列是0， 而不是没有这条记录。
 ```concept
  week   signup_number
   0	     5
   1	     0
   2	     3
   3	     2
```
 
 ### 参考资料
 
 * [MySQL最简单入门](https://www3.ntu.edu.sg/home/ehchua/programming/sql/MySQL_Beginner.html)
 * [SQL数据分析](http://www.silota.com/docs/recipes/mysql-sequential-generate-series-numbers-time.html)
 * [日历表](http://www.brianshowalter.com/calendar_tables)
 * [如何在MySQL里生成日期](https://blog.csdn.net/qiuli_liu/article/details/81707562)
 * [MySQL入门](https://popsql.com/learn-sql/mysql/how-to-have-multiple-counts-in-mysql/)