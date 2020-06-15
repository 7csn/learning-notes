# MySQL基准测试
定义：
    
* 针对系统设置的一种压力测试，重性能，忽略逻辑

基准测试：
* 直接、简单、易于比较，用于评估服务器的处理能力
* 可能不关心业务逻辑，所使用的查询和业务的真实性可以和业务环境无关

压力测试：
* 对真实的业务数据进行测试，获得真实系统所能承受的压力
* 需要针对不同主题，所使用的数据和查询也是真实用到的


#### 基准测试的目的

* 建立MySQL服务器的性能基准线
    * 确定当前MySQL服务器运行情况
 
* 模拟比当前系统更高的负载，以找出系统的扩展瓶颈
    * 增加数据库并发，观察QPS、TPS变化，确定并发量与性能最优的关系

* 测试不同的硬件、软件和操作系统配置

* 证明新的硬件设备是否配置正确

#### 基准测试的方法

* 对整个系统进行基准测试

    从系统入口进行测试（如网站Web端，手机APP前端）
    
    优点：
    
    * 能够测试整个系统的性能，包括web服务器缓存、数据库等
    
    * 能反应出系统中各个组件接口间的性能问题，体现真实性能情况
    
    缺点：
    
    * 测试设计复杂，消耗时间长

* 单独对MySQL进行基准测试

    优点：
    
    * 测试设计简单，耗时短
    
    缺点：
    
    * 无法全面了解整个系统的性能基线

#### MySQL基准测试的常见指标

* 单位时间所处理的事务数（TPS）

* 单位时间所处理的查询数（QPS）

* 响应时间

    * 平均响应时间、最小响应时间、最大响应时间、各时间所占百分比（有价值数据）

* 并发量：同时处理的查询请求的数量

#### 基准测试的步骤

计划和设计基准测试

* 对整个系统还是某一组件

* 使用什么样的数据

* 准备基准测试数据收集脚本

    * CPU使用率、IO、网络流量、状态与计数器信息等
    
        > 脚本参考图  
        > ![脚本图1](./img/03.1.png?raw=true "Get_Test_info.sh")
        > ![脚本图2](./img/03.2.png?raw=true "Get_Test_info.sh")

* 运行基准测试

* 保存及分析基准测试结果

#### 基准测试中容易忽略的问题

* 使用生产环境数据时只使用了部分数据

    * 推荐：使用数据库完全备份来测试

* 在多用户场景中，只做单用户的测试

    * 推荐：使用多线程并发测试

* 在但服务器上测试分布式应用

    * 推荐：使用相同架构进行测试

* 反复执行同一查询，容易缓存命中，无法体现真实查询性能

#### 常用基准测试工具

* mysqlslap

    * MySQL服务器自带的基准测试工具
    
    特点：
    
    * 可以模拟服务器的负载，并输出相关统计信息
    
    * 可以指定也可以自动生成查询语句
    
    常用参数说明：
    
    * --auto-generate-sql 由系统自动生成SQL脚本进行测试
    
    * --auto-generate-sql-add-autoincrement 在生成的表中增加自增ID
    
    * --auto-generate-sql-load-type 指定测试中使用的查询类型，read/write/update/mixed（默认）
    
    * --auto-generate-sql-write-number 指定初始化数据时生成的数据量，默认100行
    
    * --concurrency 指定并发线程的数量，可用逗号分隔多个
    
    * --engine 指定要测试表的存储引擎，可用逗号分隔多个
    
    * --no-drop 指定不清理测试数据
    
    * --iterations 指定测试运行的次数，和--no-drop冲突
    
    * --number-of-queries 指定每个线程执行的查询数量
    
    * --debug-info 指定输出额外的内存及CPU信息
    
    * --number-int-cols 指定测试表中包含的INI类型列的数量
    
    * --number-char-clos 指定测试表中包含的varchar类型的数量
    
    * --create-schema 制定了用于执行测试的数据库的名字
    
    * --query 用于指定自定义SQL的脚本
    
    * --only-print 并不运行测试脚本，而是把生成的脚本打印出来

* sysbench

    安装说明：
    ```shell
    > curl https://github.com/akopytov/sysbench/archive/0.5.zip
    > unzip sysbench-0.5.zip
    > ch sysbench-0.5
    > ./autogen.sh
    > ./configure --with-mysql-includes=/usr/local/mysql/include/ --with-mysql-libs=/usr/local/mysql/lib/
    > make
    ```
    
    常用参数：
    
    * --test 用于指定索要执行的测试类型
        * fileio：文件系统I/O新能测试
        * cpu：cpu性能测试
        * memory：内存性能测试
        * ./tests/db/oltp.lua
    
    * --mysql-db 用于指定执行基准测试的数据库名
    
    * --mysql-table-engine 用于指定所使用的存储引擎
    
    * --oltp-tables-count 执行测的表的数量
    
    * --oltp-table-size 指定每个表中的数据行数
    
    * --num-threads 指定测试的并发线程数量
    
    * --max-time 指定最大的测试时间
    
    * --report-interval 指定间隔多长时间输出一次统计信息
    
    * --mysql-user 指定执行测试的MySQL用户
    
    * --mysql-password 指定执行测试的MySQL用户的密码
    
    * prepare 用于准备测试数据
    
    * run 用于实际进行测试
    
    * cleanup 用于清理测试数据

#### sysbench基准测试演示实例

简单测试：
```shell
# 单核内存生成10000质数
> sysbench --test=cpu --cpu-max-prime=10000

# 查看内存
> free -m

* 查看磁盘
> df -lh

# 准备1G测试文件
> sysbench --test=fileio --file-total-size=1G prepare

# 每秒输出测试8线程文件随机读写统计信息
> sysbench --test=fileio --num-threads=8 --init-rng=on --file-total-size=1G --file-test-mode=rndrw --report-interval=1

# 单核内存生成10000质数
> sysbench --test=cpu --cpu-max-prime=10000
```

创建数据库及用户权限：
```
mysql> create database imooc;
mysql> grant all privileges on *.* to sbtest@'localhost' indentified by '123456';
```

测试：
```shell
# 生成10张表，各10000行数据
> sysbench --test=./tests/db/oltp.lua --mysql-table-engine=innodb --oltp-table-size:10000 --mysql-db=imooc --mysql-user=sbtest --mysql-password=123456 --oltp-tables-count=10 --mysql-socket=/usr/local/mysql/data/mysql.lock prepare

# 执行【基准测试的步骤】中图片相关（检测统计信息）脚本
> bash ./Get_Test_info.sh &

# 执行测试
> sysbench --test=./tests/db/oltp.lua --mysql-table-engine=innodb --oltp-table-size:10000 --mysql-db=imooc --mysql-user=sbtest --mysql-password=123456 --oltp-tables-count=10 --mysql-socket=/usr/local/mysql/data/mysql.lock run
```
