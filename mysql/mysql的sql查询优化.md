# 获取有性能问题的sql方法：  
    通过慢查询日志，分析工具mysqldumpslow  
    实时方法获取有性能问题：通过information_schema  
# 查询流程与具体内容：  
  #### 1。客户端发送请求给服务器  
  #### 2。服务器查询缓存，有缓存返回结构：  
    通过一个对大小写敏感的哈希查找实现。只能全值匹配。缓存会检查更新，会加锁。  
    所以在读写频繁的系统使用查询缓存会降低查询效率。  
  #### 3。没缓存，解析SQL、预处理、再由优化器生成执行计划    
    解析SQL：通过关键字对MYSQL语句进行解析，生成解析树。  
    预处理：检查解析树是否合法    
    优化器：优化sql语句  
  #### 4。Mysql根据执行计划，调用存储引擎的API查询    
  #### 5。返回结果    
  #### 2到5都有可能对查询造成影响    
#  如何确定查询处理各个阶段所消耗时间    
&#160;&#160;set profiling=1；show profiles；show profile for query N；  
&#160;&#160;使用performance_schema  
# 优化器的优化：
**错误优化的原因：**  
&#160;&#160;1.统计信息可能不准确，  
&#160;&#160;2.执行计划成本估算不等于实际执行计划成本（优化器不能知道页在内存，磁盘上，哪些页面需要顺序，随机读取）。    
&#160;&#160;3.mysql优化器认为的最优不是时间    
&#160;&#160;4.mysql不考虑其他并发查询，存储过程，自定义函数，可能会影响当前查询    
&#160;&#160;5.mysql也会基于一些固定规则来生成执行计划  
&#160;&#160;6.MySQL优化器所调查的可能的方案数随查询中所引用的表的数目呈指数增长。当提交的查询更大时，查询优化所花的时间会很容易地成为服务器性能的主要瓶颈。   
**可以优化的内容：**    
&#160;&#160;1.重新定义表的关联顺序     
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;根据统计信息    
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;将外连接转化为内链接：外连接等价于内链接情况    
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;使用等价变化规则：如 5 =5 and a>5  => a>5    
&#160;&#160;2.优化count，min，max：   
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;查询最小和最大，可以直接读取b树第一和最后一个    
&#160;&#160;3.将一个表达式转化为常数    
&#160;&#160;4.使用覆盖索引返回查询信息    
&#160;&#160;5.子查询优化：转化为关联查询    
&#160;&#160;6.提前终止查询：limit或一个不成立的条件    
&#160;&#160;7.对in条件优化（比or块，in中的数据先排序，二分查找来确定是否满足条件）    

**Mysql查询优化器** 
1.常量转化

    它能够对sql语句中的常量进行转化，比如下面的表达式：　WHERE col1 = col2 AND col2 = 'x';　依据传递性：如果A=B and B=C，那么就能得出A=C。所以上面的表达式mysql查询优化器能进行如下的优化：WHERE col1 = 'x' AND col2 = 'x';　对于col1 col2，只要是属于下面的操作符之一就可以进行类似的转化：　=,<,>,<=,>=,<>,<=>,LIKE
    从中我们也可以看出，对于 BETWEEN的情况是不进行转换的。这个可能与其具体的实现有关。

2.无效代码的排除

　   查询优化器会对一些无用的条件进行过滤，比如说 WHERE 0=0 AND column1='y'　　因为第一个条件是始终为true的，所以可以移除该条件，变为：WHERE column1='y'再见如下表达式：WHERE (0=1 AND s1=5) OR s1=7因为前一个括号内的表达式始终为false，因此可以移除该表达式，变为：WHERE s1=7
    一些情况下甚至可 以将整个WHERE子句去掉，见下面的表达式：WHERE (0=1 AND s1=5)我们可以看到，WHERE子句始终为FALASE，那么WHERE条件是不可能发生的。当然我们也可以讲，WHERE条件被优化掉了。
    如果一个列的定义是不允许为NULL，那么：WHERE not_null_column IS NULL该条件 是始终为false的，再看：WHERE not_null_column IS NOT NULL该条件是始终为 true的，因此这样的表达式也是可以从条件表达式中删除的。
    当然，也是有特殊情况的，比如在out join中，被定义为NOT NULL的列也可能包含NULL值。在这种情况下，IS NULL条件是被保留的。
    当然优化器没有对所有的情况进行检测，因为这实在太 复杂了。举个例子：CREATE TABLE Table1(column1 CHAR(1));
    SELECT * FROM Table1 WHERE column1 = 'Canada';尽管该条件是无效条件，优化器也不会将它移除。

3.常量计算

    如下表达式：WHERE col1 = 1 + 2转化为：WHERE col1 = 3  Mysql会对常量表达进行计算，然后将结果生成条件

4.存取类型

    当我们评估一个条件表达式，MySQL判断该表达式的存取类型。下面是一些存取类型，按照从最优到最差的顺序进行排列：
    system系统表，并且是常量表
    const  常量表
    eq_ref  unique/primary索引，并且使用的是'='进行存取
    ref  索引使用'='进行存取
    ref_or_null  索引使用'='进行存取，并且有可能为NULL
    range  索引使用BETWEEN、IN、>=、LIKE等进行存取
    index  索引全扫描
    ALL  表全扫描
    优化器根据存取类型选择合适的驱动表达式。考虑如下的查询语句：以下是引用片段：
        SELECT *　FROM Table1  WHERE indexed_column=5 AND  unindexed_column=6
    因为indexed_column拥有更好的存取类型，所以更有可能使用该表达式做为驱动表达式。这里只考虑简单的情况，不考虑特殊的情况。那么驱动表达式的意思是什么呢?考虑到这个查询语句有两种可能的执行方法:
    1) 不好的执行路径：读取表的每一行(称为“全表扫描”)，对于读取到的每一行，检查相应的值是否满足indexed_column以及 unindexed_column对应的条件。
    2) 好的执行路径：通过键值indexed_column=5查找B树，对于符合该条件的每一行，判断是否满足unindexed_column对应的条件。
    一般情况下，索引查找比全表扫描需要更少的存取路径，尤其当表数据量很大，并且索引的类型是UNIQUE的时候。因此称它为好的执行路径，使用 indexed_column列作为驱动表达式。

5.范围存取类型
    
    一些表达式可以使用索引，但是属于索引的范围查找。这些表达式通常对应的操作符是：>、>=、<、<=、IN、LIKE、 BETWEEN。
　　  对优化器而言，如下表达式：
　　  column1 IN (1,2,3)
　　  该表达式与下面的表达式是等价的：
      column1 = 1 OR column1 = 2 OR column1 = 3
    并且 MySQL也是认为它们是等价的，所以没必要手动将IN改成OR,或者把OR改成IN。
    优化器将会对下面的表达式使用索引范围查找：column1 LIKE 'x%'，但对下面的表达式就不会使用到索引了：column1 LIKE '%x'，这是因为当首字符是通配符的时候， 没办法使用到索引进行范围查找。
    对优化器而言，如下表达式：column1 BETWEEN 5 AND 7　该表达式与下面的表达式是等价的：column1 >= 5 AND column1 <= 7同样，MySQL也认为它们是等价的。
    如果需要检查过多的索引键值，优化器将放弃使用索引范围查找，而是使用全表扫描的方式。这样的情况经常出现如下的情况下：索引是多层次的二级索引，查询条件是'<'以及是'>'的情况。

6.索引存取类型
    
    考虑如下的查询语句：SELECT column1 FROM Table1;如果column1是索引列， 优化器更有可能选择索引全扫描，而不是采用表全扫描。这是因为该索引覆盖了我们所需要查询的列。　再考虑如下的查询语句：　SELECT column1,column2 FROM Table1;　　如果索引的定义如下，那么就可以使用索引全扫描：CREATE INDEX … ON Table1(column1,column2);　　也就是说，所有需要查询的列必须在索引中出现。但是如下的查询就只能走全表扫描了： select col3 from Table1;由于col3没有建立索引所以只能走全表扫描。由此其实我们的Cn表中建立的索引其实还是有一些问题的：
    PRIMARY KEY  (`CID`),
      UNIQUE KEY `IDX_CN_CNAME` (`CNAME`),
      KEY `INDEX_CN_CID_UID` (`CID`,`CUSTOMERID`),
      KEY `INDEX_CN_PRODTYPE` (`PRODTYPE`),
      KEY `INDEX_CN_P_C` (`PRODTYPE`,`CNSTATUS`),
      KEY `INDEX_CN_UID` (`CUSTOMERID`)
    比如所cid是唯一索引，由cid已经能唯一确定一条记录，那么在以cid和customerid建立索引实际上是多余的。同样，建立了prodtype和cnstatus的复合索引，再建立prodtype的索引也是有问题的，即使你使用了prodtype字段作为条件查询，也未必就会使用prodtype的索引，因为他们有着相同的前缀，故优化器根本搞不清楚你要使用哪个索引，所以，尽量避免相同的前缀的索引。

7.转换
MySQL对简单的表达式支持转换。比如下面的语法：WHERE -5 = column1转换为：　 　WHERE column1 = -5　尽管如此，对于有数学运算存在的情况不会进行转换。比如下面的语法：　WHERE 5 = -column1不会转换为：WHERE column1 = -5，所以尽量减少列上的运算，而将运算放到常量上。比如我们在写sql的时候自觉的将5= -columb1=> column1=-5;

 

8.AND
带AND的查询的格式为： AND ，考虑如下的查询语句：

WHERE column1='x' AND column2='y'　　

优化的步骤：

1) 如果两个列都没有索引，那么使用全表扫描。

　2) 否则，如果其中一个列拥有更好的存取类型(比如，一个具有索引，另外一个没有索引;再或者，一个是唯一索引，另外一个是非唯一索引)，那么使用该列作为驱动表达式。

　3) 否则，如果两个列都分别拥有索引，并且两个条件对应的存取类型是一致的，那么选择定义索引时,先定义的索引。

　 　举例如下：

　　CREATE TABLE Table1 (s1 INT,s2 INT);

　　CREATE INDEX Index1 ON Table1(s2);

　　CREATE INDEX Index2 ON Table1(s1);

　 　…

　　SELECT * FROM Table1 WHERE s1=5 AND s2=5;

　　优化器选择s2=5作为驱动表达式，因为s2上的索引是创建的时间早。

 

9         OR
带OR的查询格式为： OR ，考虑如下的查询语句：WHERE column1='x' OR column2='y'

优化器做出的选择是采用全表扫描。当然，在一些特定的情况，可以使用索引合并，这里不做阐述。如果两个条件里面设计的列是同一列，那么又是另外一种情况，考虑如下的查询语句：WHERE column1='x' OR column1='y'在这种情况下，该查询语句采用索引范围查找。

10    UNION
所有带UNION的查询语句都是单独优化的，考虑如下的查询语句：以下是引用片段：　　SELECT *  FROM Table1   WHERE  column1='x'　

UNIONALL　 SELECT * FROM Table1  WHER  column2='y'

如果column1与column2都是拥有索引 的，每个查询都是使用索引查询，然后合并结果集。

11    NOT,<>
考虑如下的表达式：　Column1<> 5从逻辑上讲，该表达式等价于下面的表达式：

Column1<5 OR column1>5　然而，MySQL不会进行这样的转换。如果你觉得使用范围查找会更好一些，应该手动地进行转换。

　　考虑如下的表达式：　WHERE NOT (column1!=5) 从逻辑上讲，该表达式等价于下面的表达式：WHERE column1=5　同样地，MySQL也不会进行这样的转换。

12    ORDER BY
一般而言，ORDER BY的作用是使结果集按照一定的顺序排序，如果可以不经过此操作就能产生顺序的结果，可以跳过该ORDER BY操作。考虑如下的查询 语句：

SELECT column1 FROM Table1 ORDER BY 'x';优化器将去除该 ORDER BY子句，因为此处的ORDER BY子句没有意义。再考虑另外的一个查询语句：SELECT column1 FROM Table1 ORDER BY column1;

在这种情况下，如果column1类上存在索引，优化器将使用该索引进行全扫描，这样产生的结果集是有序的，从而不需要进行ORDER BY操作。

再考虑另外的一个查询语句：SELECT column1 FROM Table1 ORDER BY column1+1;　　假设column1上存在索引，我 们也许会觉得优化器会对column1索引进行全扫描，并且不进行ORDER BY操作。实际上，情况并不是这样，优化器是使用column1列上的索引进行全扫表，仅仅是因为索引全扫描的效率高于表全扫描。对于索引全扫描的结果集 仍然进行ORDER BY排序操作。

13    GROUP BY
这里列出对GROUP BY子句以及相关集函数进行优化的方法：

1)      如果存在索引，GROUP BY将使用索引。

2) 如果没有索引，优化器将需要进行排序，一般情况下会使用HASH表的方法。

3) 如果情况类似于“GROUP BY x ORDER BY x”,优化器将会发现ORDER BY子句是没有必要的，因为GROUP BY产生的结果集是按照x进行排序的。

4) 尽量将HAVING子句中的条件提升中WHERE子句中。

5) 对于MyISAM表，“SELECT COUNT(*) FROM Table1;”直接返回结果，而不需要进行表全扫描。但是对于InnoDB表，则不适合该规则。补充一点，如果column1的定义是NOT NULL的，那么语句“SELECT COUNT(column1) FROM Table1;”等价于“SELECT COUNT(*) FROM Table1;”。

6) 考虑MAX()以及MIN()的优化情况。考虑下面的查询语句：以下是引用片段：
　 　SELECTMAX(column1)　FROMTable1　WHEREcolumn1<'a'; 　如果column1列上存在索引，优化器使用'a'进行索引定位，然后返回前一条记录。

7) 考虑如下的查询语句:

SELECT DISTINCT column1 FROM Table1;在特定的情况下，语句可以转化为：

　　 SELECT column1 FROM Table1 GROUP BY column1;转换的前提条件是：column1上存 在索引，FROM上只有一个单表，没有WHERE条件并且没有LIMIT条件。


# 特定SQL的查询优化：  
&#160;&#160;1.大表的更新和删除：分批处理  
&#160;&#160;2.大表结构修改：   
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;字段类型，字段宽度修改会锁表。改名可以在线修改，主从修改切换，影子表方法（新表复制旧表数据，并用同步器同步更新，修改后，替换旧表），可以用pt-online-schema-chage  
&#160;&#160;3.优化not in 和 <> 查询：   

        select customer_id,first_name,last_name,email from customer     
        where customer_id not in (select customer_id from payment)      

        select a.customer_id,a.first_name,a.last_name,a.email from customer a  
        left join payment b on a.customer_id = b.customer_id   
        where b.customer_id is null  

&#160;&#160;4.使用汇总表优化查询  
#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;汇总表：提前将要统计的数据进行汇总并记录到表按时间段分开中，定期累积更新    
