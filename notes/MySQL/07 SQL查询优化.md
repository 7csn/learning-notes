# SQL查询优化

如何获取有性能问题的SQL
* 通过用户反馈
* 通过慢查日志
* 实时获取

使用慢查询日志获取有性能问题的SQL

* slow_query_log=ON 
    
    开启慢查日志，默认off；可通过脚本定时开关
* show_query_log_file   
    
    指定慢查日志的存储路径文件，默认保存在MySQL数据目录中；建议数据和日志分开存放
* long_query_time

    指定记录慢查日志SQL执行时间的阈值，单位秒，可设置小数，精确到微秒；通常0.001即1毫秒；注意日志增长，磁盘空间问题
    
    慢查询会记录超时的SQL：查询语句、数据修改语句、已回滚语句
* log_queries_not_using_indexes

    是否记录未使用索引的SQL 

慢查询日志基本信息：

![慢查日志详情](./img/07.1.png?raw=true "慢查日志详情")

常用慢查询日志分析工具：
* mysqldumpslow

    ![mysqldumpslow](./img/07.2.png?raw=true "mysqldumpslow")  
    ![mysqldumpslow](./img/07.3.png?raw=true "mysqldumpslow")

* pt-query-digest（推荐）

    `> pt-query-digest --explain h=IP,u=用户,p=密码 日志路径`
    
如何实时获取有性能问题的SQL:

    ```
    # 查询耗时的SQL
    mysql> SELECT id,user,host,db,command,time,state,info FROM information_schema.processlist WHERE time >= 60;
    ```
    
    可以周期性查询上述语句达到监控目的


MySQL服务器处理查询请求的整个过程：

* 客户端发送SQL请求给服务器

* 服务器检查是否可以在查询缓存中命中该SQL
    * 通过对大小写敏感的哈希查找实现，SQL必须完全一致
    * 会对缓存加锁，对读写频繁的系统可能会降低查询处理的效率
    
    相关参数：
    * query_cache_type：设置查询缓存是否可用
        * on
        * off
        * demand 表示只有在查询语句中使用sql_cache和sql_no_cache来控制是否需要缓存 
    * query_cache_size：设置查询缓存的内存大小
    * query_cache_limit：设置查询缓存可用存储的最大值
    * query_cache_wlock_invalidate：设置数据表被锁后是否返回缓存中的数据，默认关闭
    * query_cache_min_res_unit：设置查询缓存分配的内存块最小单位
    
    读写频繁的系统，建议：

    ```
    query_cache_type=off
    query_cache_size=0
    ```

* 服务端执行SQL解析，预处理，再由优化器生成对应的执行计划

    解析SQL：对SQL语句进行验证，验证是否使用了正确的关键字，关键字顺序是否正确等，解析成对应的“解析树”
    
    预处理：检查查询中表和数据列是否存在，名字或别名是否有歧义等
    
    查询优化器：生成查询计划
    
    造成MySQL生成错误的执行计划的原因：
    * 统计信息不准确
    * 执行计划中的成本估算不等同于实际的执行计划的成本
        * MySQL服务层不知道哪些页面在内存中，哪些页面在磁盘上，哪些页面需要顺序读取，哪些需要页面随机读取
    * MySQL优化器锁认为的最优可能和我们所认为的最优不一样
        * 基于其成本模型选择最优的执行计划
    * MySQL从不考虑其它并发的查询，可能会影响当前查询的速度
    * MySQL有时候会基于一些固定的规则来生成执行计划
    * MySQL不会考虑不受其控制的成本
        * 存储过程
        * 用户自定义函数
    
    优化器作用：
    * 重新定义表的关联顺序
    * 将外连接转化成内连接
    * 使用等价变换规则
    * 优化count()、min()和max()
    * 将一个表达式转化未常数表达式
    * 使用等价变化规则
    * 子查询优化（转化为关联查询）
    * 提前终止查询
    * 对in()条件进行优化

* 根据执行计划，调用存储引擎API来查询数据

* 将结果返回给客户端

如何确定查询处理各个阶段所消耗的时间：
* 使用profile

    ```
    # 启动profile（当前session有效）
    mysql> set profiling=1;
    
    # 查询各语句持续时间
    mysql> show profiles;
    
    # 查询指定语句详情
    mysql> show profile for query 查询ID;
    
    # 查询指定语句详情（包含cpu信息）
    mysql> show profile cpu for query 查询ID;
    ``` 
* 使用performance_schema（5.5+）

    5.6+推荐使用
    
    启用performance_schema，启用后对数据库全局有效：
    
    ![启用performance_schema](./img/07.4.png?raw=true "启用performance_schema")
    
    ```
    mysql> update setup_instruments set enabled='YES',timed='YES' where name like 'stage%';
    
    mysql> update setup_consumers set enabled='YES' where name like 'events%';
    ```
    
    查询SQL执行详情：
    
    ![使用performance_schema](./img/07.5.png?raw=true "使用performance_schema")
    
    ```
    mysql> SELECT a.THREAD_ID,SQL_TEST,C.EVENT_NAME,(C.TIMER_END-C.TIMER_START)/1000000000 AS 'DURATION(ms)' FROM events_statements_history_long a JOIN
    threads b ON a.THREAD_ID=b.THREAD_ID JOIN events_stages_history_long c ON c.THERAD_ID=b.THREAD_ID AND c.EVENT_ID BETWEEN a.EVENT_ID AND a.END_EVENT_ID WHERE b.PROCESSLIST_ID=CONNECTION_ID() AND a.EVENT_NAME='statement/sql/select' ORDER BY a.THREAD_ID,c.EVENT_ID;
    ```

特定SQL的查询优化：

* 大表的更新和删除

    ![大表的更新和删除](./img/07.6.png?raw=true "大表的更新和删除")
    
    对表中的列的字段类型或字段宽度进行修改，还是会锁表，无法解决主从延迟的问题
    
    如何修改大表结构：
    * 利用主从复制架构，先在从服务器修改，再主从切换，最后修改原主服务器的大表；存在风险
    * 主服务器建立新表，导入一份旧表数据；在旧表加触发器，触发更新新表；旧表加排他锁，重命名新表名，删除旧表。只在重命名时有短暂锁，尽量减少主从延迟。
        
        工具实现：
        
        ![修改大表结构](./img/07.7.png?raw=true "修改大表结构")

* 优化not in和<>查询

    not in优化案例：
    
    ![not in优化案例](./img/07.8.png?raw=true "not in优化案例")

* 使用汇总表优化查询

    汇总表就是提前以要套机的数据进行汇总并记录到表中，以备后续的查询使用
    
    ![汇总表优化查询案例](./img/07.9.png?raw=true "汇总表优化查询案例")
