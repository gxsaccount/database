# 大表 #  
    查：很难在有限时间内查出需要的数据，慢查询  
    删：
    建立索引：索引建立时间很长，5.5会锁表，5.6主从延迟   
    修改表结构：长期锁表，主从延迟  
处理：
    分库分表：
        难点：
            分表主键选择
            分表后跨分区数据的查询和统计
    历史数据归档：
        难点：
            归档时间点选择
            归档操作（删除大量的不会访问的数据，可能会造长）
        
历史数据归档
# 大事务 #  
运行时间比较长，操作数据比较多（都会被行锁锁定）  
风险：锁定太多数据，造成大量的阻塞和锁超时  
      回滚所需时间长
      主从延迟，从服务器只有等到主库执行完事务，并将日志记录到日志文件才能同步  
处理：
    分批次处理
    事务移除不必要的select操作
