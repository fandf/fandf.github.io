---
layout: post
title: MySQL Explain详解与索引最佳实践
date: 2023-03-03
tags: [mysql]
description: 欲买桂花同载酒，终不似，少年游
---

## 新建示例表和数据
```mysql
-- 演员表
DROP TABLE IF EXISTS `actor`; 
CREATE TABLE `actor` ( 
`id` int(11) NOT NULL, 
`name` varchar(45) DEFAULT NULL, 
`update_time` datetime DEFAULT NULL, 
PRIMARY KEY (`id`) 
) ENGINE=InnoDB DEFAULT CHARSET=utf8; 

INSERT INTO `actor` (`id`, `name`, `update_time`) VALUES 
(1,'a', NOW()), 
(2,'b', NOW()), 
(3,'c', NOW()); 

-- 电影表
DROP TABLE IF EXISTS `film`; 
CREATE TABLE `film` (`id` int(11) NOT NULL AUTO_INCREMENT, 
`name` varchar(10) DEFAULT NULL, 
PRIMARY KEY (`id`), 
KEY `idx_name` (`name`) 
) ENGINE=InnoDB DEFAULT CHARSET=utf8; 

INSERT INTO `film` (`id`, `name`) VALUES
(3,'film0'),
(1,'film1'),
(2,'film2'); 

-- 演员电影表
DROP TABLE IF EXISTS `film_actor`; 
CREATE TABLE `film_actor` ( 
`id` int(11) NOT NULL, 
`film_id` int(11) NOT NULL, 
`actor_id` int(11) NOT NULL, 
`remark` varchar(255) DEFAULT NULL, 
PRIMARY KEY (`id`), 
KEY `idx_film_actor_id` (`film_id`,`actor_id`) 
) ENGINE=InnoDB DEFAULT CHARSET=utf8; 

INSERT INTO `film_actor` (`id`, `film_id`, `actor_id`) VALUES 
(1,1,1),
(2,1,2),
(3,2,1);
```

## explain列详解
> explain select * from actor;

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d68d33e34bbc424ba79cc3a8a1671c0e~tplv-k3u1fbpfcp-watermark.image)
### id列
> id列的编号是 select 的序列号.有几个 select 就有几个id，并且id的顺序是按 select 出现的顺序增长的。 `id列越大执行优先级越高，id相同则从上往下执行，id为NULL最后执行`。

### select_type列
> 1）simple：简单查询。查询不包含子查询和union  
> 2）primary：复杂查询中最外层的 select   
> 3）subquery：包含在 select 中的子查询（不在 from 子句中）   
> 4）derived：包含在 from 子句中的子查询。MySQL会将结果存放在一个临时表中，也称为派生表（derived的英文含 义）
```mysql
set session optimizer_switch='derived_merge=off'; #关闭mysql5.7新特性对衍生表的合 并优化
explain select (select 1 from actor where id = 1) from (select * from film where id = 1) t;
set session optimizer_switch='derived_merge=on'; #还原默认配置
 ```
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f5f712efa0654875957895fea88defbc~tplv-k3u1fbpfcp-watermark.image)
> 5）union：在 union 中的第二个和随后的 select
```mysql
explain select 1 union all select 1;
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e64f1153db749b89bd2517673bc55d4~tplv-k3u1fbpfcp-watermark.image)
### table列
> 这一列表示 explain 的一行正在访问哪个表。   
> 当 from 子句中有子查询时，table列是 <derivenN> 格式，表示当前查询依赖 id=N 的查询，于是先执行 id=N 的查询。  
> 当有 union 时，UNION RESULT 的 table 列的值为<union1,2>，1和2表示参与 union 的 select 行id。
### partitions列
> 表分区 （现在应该很少有人做表分区了吧，个人见解）
### type列
>这一列表示关联类型或访问类型，即MySQL决定如何查找表中的行，查找数据行记录的大概范围。   
> 依次从最优到最差分别为：system > const > eq_ref > ref > range > index > ALL   
> 一般来说，得保证查询达到range级别，最好达到ref。

#### 1) NULL：mysql能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引。
> 例如：在索引列中选取最小值，可以单独查找索引来完成，不需要在执行时访问表
```mysql
explain select min(id) from film;
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3cae328f77f142309881e09d66c0c3ab~tplv-k3u1fbpfcp-watermark.image)
#### 2) const, system
> mysql能对查询的某部分进行优化并将其转化成一个常量。用于 primary key 或 unique key 的所有列与常数比较时，所以表最多有一个匹配行，读取1次，速度比较快。system是 const的特例，表里只有一条元组匹配时为system
```mysql
set session optimizer_switch='derived_merge=off'; #关闭mysql5.7新特性对衍生表的合 并优化
explain select * from (select * from film where id = 1) tmp;
set session optimizer_switch='derived_merge=on'; #还原默认配置
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa2e80549ce541e28bfd3a340501deb4~tplv-k3u1fbpfcp-watermark.image)
#### 3) eq_ref
> primary key 或 unique key 索引的所有部分被连接使用 ，最多只会返回一条符合条件的记录。这可能是在 const 之外最好的联接类型了，简单的 select 查询不会出现这种 type。
```mysql
explain select * from film_actor left join film on film_actor.film_id = film.id
```
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b56fdb885d24e669e028b8d4ee813ab~tplv-k3u1fbpfcp-watermark.image)

#### 4) ref
> 相比 eq_ref，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，索引要和某个值相比较，可能会找到多个符合条件的行。
1. 简单 select 查询，name是普通索引（非唯一索引）
```mysql
explain select * from film where name = 'film1';
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c730c353205b4c7aaeb743a1a5e15880~tplv-k3u1fbpfcp-watermark.image)
2. 关联表查询，表film_actor中idx_film_actor_id是film_id和actor_id的联合索引，这里使用到了film_actor的左边前缀film_id部分。
```mysql
explain select film_id from film f left join film_actor fa on f.id = fa.film_id;
```
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b54e0f3c832047adbd1b091539035c2f~tplv-k3u1fbpfcp-watermark.image)
#### 5) range
> 范围扫描通常出现在 in(), between ,> ,<, >= 等操作中。使用一个索引来检索给定范围的行。
```mysql
explain select * from actor where id > 1;
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce866c72da724f688b88bce55e3ad562~tplv-k3u1fbpfcp-watermark.image)
#### 6) index
> 扫描全索引就能拿到结果，一般是扫描某个二级索引，这种扫描不会从索引树根节点开始快速查找，而是直接对二级索引的叶子节点遍历和扫描，速度还是比较慢的，这种查询一般为使用覆盖索引，二级索引一般比较小，所以这种通常比ALL快一些。
```mysql
explain select * from film;
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b8b6d6e6d574185b62458d25f01c9e2~tplv-k3u1fbpfcp-watermark.image)
#### 7) ALL
>即全表扫描，扫描你的聚簇索引的所有叶子节点。通常情况下这需要增加索引来进行优化了。
```mysql
explain select * from actor;
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa77d382022a48df82a0073163646997~tplv-k3u1fbpfcp-watermark.image)
### possible_keys列
> 这一列显示查询可能使用哪些索引来查找。  
> explain 时可能出现 possible_keys 有列，而 key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql认为索引对此查询帮助不大，选择了全表查询。   
> 如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查 where 子句看是否可以创造一个适当的索引来提 高查询性能，然后用 explain 查看效果。

### key列
> 这一列显示mysql实际采用哪个索引来优化对该表的访问。 如果没有使用索引，则该列是 NULL。如果想强制mysql使用或忽视possible_keys列中的索引，在查询中使用 force index、ignore index。
### key_len
> 这一列显示了mysql在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些列。
举例来说，film_actor的联合索引 idx_film_actor_id 由 film_id 和 actor_id 两个int列组成，并且每个int是4字节。通过结果中的key_len=4可推断出查询使用了第一个列：film_id列来执行索引查找。
```mysql
explain select * from film_actor where film_id = 2;
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f0a8e23bbff4c3bad0348cb449c36ea~tplv-k3u1fbpfcp-watermark.image)
key_len计算规则如下：
- 字符串，char(n)和varchar(n)，5.0.3以后版本中，n均代表字符数，而不是字节数，如果是utf-8，一个数字 或字母占1个字节，一个汉字占3个字节
    - char(n)：如果存汉字长度就是 3n 字节
    - varchar(n)：如果存汉字则长度是 3n + 2 字节，加的2字节用来存储字符串长度，因为varchar是变长字符串
- 数值类型
    - tinyint：1字节
    - smallint：2字节
    - int：4字节
    - bigint：8字节
- 时间类型
    - date：3字节
    - timestamp：4字节
    - datetime：8字节
- 如果字段允许为 NULL，需要1字节记录是否为 NULL
  `索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。`
### ref列
> 这一列显示了在key列记录的索引中，表查找值所用到的列或常量，常见的有：const（常量），字段名（例：film.id）
### rows列
> 这一列是mysql估计要读取并检测的行数，注意这个不是结果集里的行数。
### filtered列
> rows * filtered/100 可以估算出将要和 explain 中前一个表进行连接的行数（前一个表指 explain 中的id值比当前表id值小的表）。
### Extra列
> 这一列展示的是额外信息。常见的重要值如下：
#### 1）Using index
使用覆盖索引 覆盖索引定义：mysql执行计划explain结果里的key有使用索引，如果select后面查询的字段都可以从这个索引的树中 获取，这种情况一般可以说是用到了覆盖索引，extra里一般都有using index；覆盖索引一般针对的是辅助索引，整个 查询结果只通过辅助索引就能拿到结果，不需要通过辅助索引树找到主键，再通过主键去主键索引树里获取其它字段值
#### 2）Using where
使用 where 语句来处理结果，并且查询的列未被索引覆盖
#### 3）Using index condition：
查询的列不完全被索引覆盖，where条件中是一个前导列的范围；
```mysql
explain select * from film_actor where film_id > 1;
```
#### 4）Using temporary
mysql需要创建一张临时表来处理查询。出现这种情况一般是要进行优化的，首先是想到用索引来优化。
1. actor.name没有索引，此时创建了张临时表来distinct
```mysql
explain select distinct name from actor; 
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a194d75994646d2805b8d83d69d1a0c~tplv-k3u1fbpfcp-watermark.image)
2. film.name建立了idx_name索引，此时查询时extra是using index,没有用临时表
```mysql
explain select distinct name from film;
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b99d460bf8eb4415b9542ab3a500229c~tplv-k3u1fbpfcp-watermark.image)
#### 5）Using filesort
将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序。这种情况下一 般也是要考虑使用索引来优化的。
1. actor.name未创建索引，会浏览actor整个表，保存排序关键字name和对应的id，然后排序name并检索行记录
```mysql
explain select * from actor order by name;
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b58cc34494ef4d919cf9b26ea8603b3b~tplv-k3u1fbpfcp-watermark.image)
2. film.name建立了idx_name索引,此时查询时extra是using index
```mysql
explain select * from film order by name;
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7df32a57295e4a5b89413dee3ebeab6d~tplv-k3u1fbpfcp-watermark.image)
#### 6）Select tables optimized away
使用某些聚合函数（比如 max、min）来访问存在索引的某个字段
```mysql
explain select min(id) from film;
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aad43c62c40f4beb95ecec225e826088~tplv-k3u1fbpfcp-watermark.image)
## 索引最佳实践
```mysql
CREATE TABLE `employees` ( 
`id` int(11) NOT NULL AUTO_INCREMENT, 
`name` varchar(24) NOT NULL DEFAULT '' COMMENT '姓名', 
`age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄', 
`position` varchar(20) NOT NULL DEFAULT '' COMMENT '职位', 
`hire_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间', 
PRIMARY KEY (`id`), 
KEY `idx_name_age_position` (`name`,`age`,`position`) USING BTREE 
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT='员工记录表'; 

INSERT INTO employees(name,age,position,hire_time) VALUES('LiLei',22,'manager',NOW()); 
INSERT INTO employees(name,age,position,hire_time) VALUES('HanMeimei', 23,'dev',NOW()); 
INSERT INTO employees(name,age,position,hire_time) VALUES('Lucy',23,'dev',NOW());
```
### 1.全值匹配
```mysql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei';
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40129719cf57402e969a2ca8517347be~tplv-k3u1fbpfcp-watermark.image)

```mysql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' and age = 22;
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8148b12185824592800e3fb4a5f5b73d~tplv-k3u1fbpfcp-watermark.image)
```mysql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' and age = 22 and position ='manager';
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74b39a248253469dacbb587fb2859589~tplv-k3u1fbpfcp-watermark.image)
### 2.最左前缀法则
如果索引了多列，要遵守最左前缀法则。指的是查询从索引的最左前列开始并且不跳过索引中的列。
```mysql
EXPLAIN SELECT * FROM employees WHERE name = 'Bill' and age = 31; 
EXPLAIN SELECT * FROM employees WHERE age = 30 AND position = 'dev'; 
EXPLAIN SELECT * FROM employees WHERE position = 'manager';
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/869135c815fe414e8001e380747f9b26~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80ad463d2db14e0d85dcf53e1e55c49c~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8fca601e42514ef598e36cf7788c4513~tplv-k3u1fbpfcp-watermark.image)
### 3.不在索引列上做任何操作
（计算、函数、（自动or手动）类型转换），会导致索引失效而转向全表扫描  
给hire_time增加一个普通索引：
```mysql
ALTER TABLE `employees` ADD INDEX `idx_hire_time` (`hire_time`) USING BTREE ; 
EXPLAIN select * from employees where date(hire_time) ='2018‐09‐30';
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b708648103e4a7396b3f9480199e6ba~tplv-k3u1fbpfcp-watermark.image)
转化为日期范围查询，有可能会走索引：

```mysql
EXPLAIN select * from employees where hire_time >='2021‐08‐09 00:00:00' and hire_time <='2021‐08‐20 23:59:59';  
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/598151a00340469bb75126dd34aff801~tplv-k3u1fbpfcp-watermark.image)

还原索引最初状态
```mysql
ALTER TABLE `employees` DROP INDEX `idx_hire_time`;
```

### 4.存储引擎不能使用索引中范围条件右边的列
```mysql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age = 22 AND position ='manager'; 
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age > 22 AND position ='manager';
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3782ed164704f1d90d99249a36e6791~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ce28c07069840ab99278f485ce741a1~tplv-k3u1fbpfcp-watermark.image)
### 5.尽量使用覆盖索引（只访问索引的查询（索引列包含查询列）），减少 select * 语句
```mysql
EXPLAIN SELECT name,age FROM employees WHERE name= 'LiLei' AND age = 23 AND position ='manager';
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2304f0c975824f84be2840dd61c56f6f~tplv-k3u1fbpfcp-watermark.image)
```mysql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age = 23 AND position ='manager';
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6cbfa44c426474e82002680cbad066a~tplv-k3u1fbpfcp-watermark.image)
### 6.mysql在使用不等于（!=或者<>），not in ，not exists 的时候无法使用索引会导致全表扫描 < 小于、 > 大于、 <=、>= 这些，mysql内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引
```mysql
EXPLAIN SELECT * FROM employees WHERE name != 'LiLei';
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e54c566e0acc4ec68d7bab6a2eb231d4~tplv-k3u1fbpfcp-watermark.image)
### 7.is null,is not null 一般情况下也无法使用索引
```mysql
EXPLAIN SELECT * FROM employees WHERE name is null;
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4211e7afc9194d879b7b66ead615d3a0~tplv-k3u1fbpfcp-watermark.image)
### 8.like以通配符开头（'$abc...'）mysql索引失效会变成全表扫描操作
```mysql
EXPLAIN SELECT * FROM employees WHERE name like '%Lei';
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ff53ebb347e41cc8e453da59b2a741d~tplv-k3u1fbpfcp-watermark.image)
```mysql
EXPLAIN SELECT * FROM employees WHERE name like 'Lei%';
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e45184ad0ef44c32ae15fbc1381b63c9~tplv-k3u1fbpfcp-watermark.image)
问题：解决like'%字符串%'索引不被使用的方法？  
a）使用覆盖索引，查询字段必须是建立覆盖索引字段
```mysql
EXPLAIN SELECT name,age,position FROM employees WHERE name like '%Lei%';
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/13d7cc3902884d85b98933d1f3a2da2e~tplv-k3u1fbpfcp-watermark.image)
b）如果不能使用覆盖索引则可能需要借助搜索引擎(es等)
### 9.字符串不加单引号索引失效
```mysql
EXPLAIN SELECT * FROM employees WHERE name = '1000'; 
EXPLAIN SELECT * FROM employees WHERE name = 1000;
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/044112e84eb14c82965bef7e2b8f7307~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/113ac0acb7b546048e68f29267dbbc08~tplv-k3u1fbpfcp-watermark.image)
### 10.少用or或in，用它查询时，mysql不一定使用索引，mysql内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引，详见范围查询优化
```mysql
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' or name = 'HanMeimei';
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64dcb9f683e34de4bcbe37b30d36a667~tplv-k3u1fbpfcp-watermark.image)
### 11.范围查询优化
给年龄添加单值索引
```mysql
ALTER TABLE `employees` ADD INDEX `idx_age` (`age`) USING BTREE ; 
explain select * from employees where age >=1 and age <=2000;
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/088b0bd77d83482f9d26302795a6cadf~tplv-k3u1fbpfcp-watermark.image)
没走索引原因：mysql内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引。比如这个例子，可能是 由于单次数据量查询过大导致优化器最终选择不走索引   
优化方法：可以将大的范围拆分成多个小范围
```mysql
explain select * from employees where age >=1 and age <=1000; 
explain select * from employees where age >=1001 and age <=2000;
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ea7681a8cf64207b02baeee34a64ac9~tplv-k3u1fbpfcp-watermark.image)

还原最初索引状态
```mysql
ALTER TABLE `employees` DROP INDEX `idx_age`;
```

## 索引使用总结
假设index(a, b, c)

where语句   |   索引是否被使用
|:---|:--------|
| where a=3  | Y,使用到a |
| where a=3 and b=4   | Y,使用到a,b |
| where a=3 and b=4 and c=5  | Y,使用到a,b,c |
| where b=3 或者 where b=4 and c=5 或者 where c=5   | N |
| where a=3 and c=5  | Y,使用到a,但是c不可以 |
| where a=3 and b>4 and c=5  | Y, 使用到a,b,但是c不可以 |
| where a=3 and b like 'k%' and c=5 | Y,使用到a,b,c |
| where a=3 and b like '%k' and c=5 | Y,只使用到a |
| where a=3 and b like '%k%' and c=5  | Y,只使用到a |
| where a=3 and b like 'k%kk%' and c=5 |Y,使用到a,b,c |

like KK%相当于=常量，%KK和%KK% 相当于范围
