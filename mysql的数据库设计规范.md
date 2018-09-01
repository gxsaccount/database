# 数据库命名规范 #  
略  
# 数据库基本设计规范 #
1.尽量使用innodb  
2.统一使用UTF-8  
3.所有表和字段需要加注释（comment）  
4.单表数据量建议在500万以内  
&emsp;&emsp;使用历史数据归档，分库分表等手段  
5.行：谨慎使用MySQL分区表  
&emsp;&emsp;根据表的不同分区文件，可以存储在不同磁盘阵列上？  
&emsp;&emsp;谨慎选择分区键，跨分区查询效率可能更低  
&emsp;&emsp;建议采用物理分表的方式管理大数据  
6.列：尽量做到冷热数据分离，减小表的宽度  
&emsp;&emsp;减少磁盘IO，保证热数据的内存命中率，避免读入冷数据  
&emsp;&emsp;经常使用的列放到一个表中  
7，禁止预留字段  
&emsp;&emsp;添加字段和修改字段前者消耗更小  
8.禁止在数据库中存储图片或文件，存地址信息即可  
# 数据库索引设计规范  
1.单张表索引最好不要超过5个  
2.主键必须有，不使用更新频繁和可重复的列，如UUID，MD5，HASH。  
3.常见索引建议：  
&emsp;&emsp;select，update，delete的where从句中  
&emsp;&emsp;order by，group by，distinct的字段  
&emsp;&emsp;多表join的关联列  
4.选择索引顺序：  
&emsp;&emsp;区分度高，字段小，使用频繁的的列放左侧  
&emsp;&emsp;避免重复和冗余索引（peimary key(id),index(id)和index(a,b,c),index(a,b)）  
5.对于频繁的查询优先考虑使用覆盖索引（所有列都在索引中）  
&emsp;&emsp;可以避免二次查询（查了非聚集索引（查到主键），还得从主键查出数据）  
&emsp;&emsp;可以将随机IO变为顺序IO加快查询效率，因为覆盖索引是按照键值顺序储存。  
6.尽量避免使用外键，避免参照完整性，建议在业务端实现  
&emsp;&emsp;外键一定会建立索引，会影响夫表和子表的写  
#数据库字段设计规范  
1.优先选择符合存储需要最小的数据类型（过大的长度会消耗更多内存）  
&emsp;&emsp;将字符串转化为数字类型存储  
&emsp;&emsp;优先选者unsigned数据类型（表示范围更广）  
&emsp;&emsp;VARCHAR尽量使用  
2.避免使用TEXT、BLOB数据类型  
&emsp;&emsp;内存表？不能使用TEXT、BLOB。只能使用磁盘表，且是前缀索引，不能有默认值。会有二次类型。建议将TEXT字段放在扩展表中。  
3.避免使用ENUM数据类型  
&emsp;&emsp;修改麻烦：必须使用alter数据  
&emsp;&emsp;enum类型的order by操作低效，需要转化为字符串再排序。  
4.尽量所有列定义为not null  
&emsp;&emsp;索引null列需要额外空间保存列为空还是非空，占用更多空间  
&emsp;&emsp;进行比较和计算时要对null做特别处理。  
5.时间，TIMESTAMP使用int存储时间，有int的大小限制。超出限制需要datetime  
6.财务相关的需要使用decimal精确计算，所占空间根据宽度定义，范围大于bigint大  
# 数据库SQL开发规范  
1.建议使用预编译语句  
&emsp;&emsp;解析一次可以重复使用  
&emsp;&emsp;使用传递参数即可，节省网络带宽  
&emsp;&emsp;一定程度防止sql注入  
2.避免数据类型隐式转化  
&emsp;&emsp;可能使索引失效  
3.充分使用索引  
&emsp;&emsp;避免使用前缀百分号 like ‘%123’，索引失效  
&emsp;&emsp;一个sql只能用到复合索引中的一列进行范围查询，如（a，b，c），在a上范围，则b，c不会再被用到  
&emsp;&emsp;使用left join或not exists 来优化not in ，索引失效  
4.禁止跨库查询  
&emsp;&emsp;为数据库迁移和分库留出余地  
&emsp;&emsp;降低业务耦合度  
&emsp;&emsp;避免权限大而产生的安全风险  
&emsp;&emsp;禁止select *,消耗跟多cpu，IO和网络带宽，无法使用覆盖索引，减少表结构变更带来影响  
&emsp;&emsp;禁止使用不含字段列表的insert语句  
&emsp;&emsp;避免使用子查询，可以优化为join  
&emsp;&emsp;&emsp;&emsp;子查询的结果集会存在临时表，没有索引，慢查询，且会消耗cpu和io  
&emsp;&emsp;避免关联太多的表  
&emsp;&emsp;&emsp;&emsp;每join一个表会多占用一份关联缓存  
&emsp;&emsp;&emsp;&emsp;会产生临时表操作，影响查询效率  
&emsp;&emsp;&emsp;&emsp;建议不要超过5个，最多61个  
&emsp;&emsp;&emsp;&emsp;对于读，减少数据库的交互次数。批量更适合批量操作，且可以提高效率（如同时增加和修改列，两个操作合并为一个只需要一个临时表）  
&emsp;&emsp;使用in代替or，or几乎不能使用索引  
&emsp;&emsp;禁止使用order by rand(),这回将所有符合条件的数据加载到内存，建议用程序获取一个随机数，去从数据库中获取随机数  
&emsp;&emsp;避免where从句禁止对列进行函数转化和计算，索引失效  
&emsp;&emsp;不需要distinst时，使用union all而不是union，因为union时生成临时表并去重  
&emsp;&emsp;拆分大SQL为小SQL  
&emsp;&emsp;&emsp;&emsp;mysql一个sql只能使用一个cpu计算，拆分后可以使用多个cpu并行计算。  
# 数据库操作行为规范  
&emsp;&emsp;对于写，超过100万行，要分批多次写。  
&emsp;&emsp;&emsp;&emsp;造成主从延迟  
&emsp;&emsp;&emsp;&emsp;产生大量日志  
&emsp;&emsp;&emsp;&emsp;避免大事务，避免长时间锁定  
&emsp;&emsp;&emsp;&emsp;大表修改表结构使用pt-online-schema-change,影子表操作  
&emsp;&emsp;&emsp;&emsp;禁止为程序使用账号的赋予super权限 ，遵循权限最小原则，不能跨库，程序账号不能有dorp权限  
