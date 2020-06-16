# MySQL高可用架构设计

#### MySQL复制功能

#### 复制解决了什么问题
* 实现在不同服务器上的数据分布
    * 利用二进制日志增量进行
    * 不需要太多的带宽
    * 基于行的复制在进行大批量更改时，会对带宽带来压力（跨IDC环境）
    * 应该分批进行
* 实现数据读取的负载均衡
    * 需要其它组件配合完成
    * 利用DNS轮询的方式把程序的读连接到不同的备份数据库
    * 使用LVS，haproxy这样的代理方式
    * 非共享架构，同样的数据据分布在多台服务器上
* 增强了数据安全性
    * 利用备库的备份来减少主库负载
    * 复制并不能代替备份
    * 方便进行数据库高可用架构的部署，避免MySQL单点失败
* 实现数据库高可用和故障切换
* 实现数据库在线升级

#### MySQL日志
* 服务器层日志
    * 二进制日志（成功执行的）：对数据库的修改，对表结构的修改，对表数据的增删改查
    * 慢查日志
    * 通用日志
* 存储引擎层日志
    * Innodb重做日志
    * Innodb回滚日志

#### MySQL二进制日志

官方提供binlog工具：mysqlbinlog

查看二进制日志文件：`> mysqlbinlog ./sql_log/bin_log名.编号 # 可以加参数如-vv`

查看当前二进制日志格式：`mysql> show variables like 'binlog_format';`

二进制日志格式：
* 基于段的格式 binlog_format=STATEMENT 5.7前默认

    优点：
    * 日志记录行相对较小，节约磁盘及网络I/O
        
    缺点：
    * 必须记录上下文信息，保证语句在从服务器上执行结构和主服务器相同
    * 特定函数如UUID()，user()这些非确定性函数无法复制
    * 可能会照成复制的主备服务器数据不一致
* 基于行的格式 binlog_format=ROW 5.7+默认

    优点：
    * 主从复制更安全
    * 对每行数据的修改比基于段的复制高效
    * 有利于恢复数据
    
    缺点：
    * 记录日志类较大 binlog_row_image=[FULL|MINIMAL|NOBLOB] # 全行|修改列|blob、text列无改不记录
* 混合日志格式 binlog_format=MIXED 

    特点：
    * 根据SQL语句由系统决定基于行或段
    * 数据量的大小由多执行的SQL语句决定

推荐使用基于行或混合日志格式，基于行，设置minimal

修改binlog格式：

```
# 设置行格式
mysql> set session binlog_format=row;

# 设置记录字段内容
mysql> set session binlog_row_image=minimal;

# 刷新binlog文件
mysql> flush logs;

# 查看binlog文件
mysql> show binary logs; 
```

#### 二进制日志格式对复制的影响

* 基于SQL语句的复制（SBR）

    优点：
    * 生成日志量少，节约网络传输I/O
    * 并不强制要求主从数据库的表定义完全相同
    * 比基于行的复制方式更灵活
    
    缺点：
    * 对于非确定性事件，无法保证主从复制一致性
    * 对于存储换成，触发器，自定义函数进行的修改也可能造成数据不一致
    * 比基于行的复制方式在主从上执行时需要更多的行锁

* 基于行的复制（RBR）

    优点：
    * 可应用于任何SQL的复制，包括非确定函数，存储过程等
    * 可减少从库锁的使用
    
    缺点：
    * 要求主从数据库的表结构相同，否则可能中断复制
    * 无法在从库单独执行触发器

* 混合模式

#### MySQL复制工作方式
1. 主库将变更写入二进制日志
2. 从读取主的二进制日志变更并写入到relay_log中
3. 在从上重放relay_log中的日志
    * 基于段的日志是在从库上重新执行SQL
    * 基于行的日志是在从库上数据行直接修改

#### 基于日志点复制
1. 在主服务器建立复制账号
    ```
    # 创建用户
    mysql> create user '用户名'@'ip段' indentified by '密码';
    
    # slave授权
    mysql> grant replication slave on *.* to '用户名'@'ip段';
    ```
2. 配置主数据库服务器

    ```
    # 开启二进制日志，并指定名称
    bin_log=mysql-bin
    
    # 唯一，可使用主机ip后几段
    server_id=100
    ```
3. 配置从数据库服务器

    ```
    # 开启二进制日志，并指定名称
    bin_log=mysql-bin
    
    # 唯一，可使用主机ip后几段
    server_id=101
    
    # 中继日志名
    relay_log=mysql-relay-bin
    
    # 从库做为其它库主库时配置，将中继日志写入到二进制日志中
    log_slave_updates=on
    
    # 阻止任何没有slave权限的用户，对开启slave的服务器进行写操作
    read_only=on
    ```
4. <a id='backup'>初始化从服务器数据</a>

    主库备份：
    
    * 自带mysqldump工具（需要加锁）
    
        ```
        # --single-transaction 保证事务一致性
        # --lock-all-tables 混合Innodb和Myisam
        # --master-data 记录主库当前二进制文件及配置信息
        # --tirggers 导出触发器
        # --routines 导出存储过程及自定义函数
        # --all-databases 所有数据库【主从版本一致，则备份业务数据库】
        > mysqldump --master-data --single-transaction --tirggers --routines --all-databases --uroot -p > 备份名.sql
        
        # 输入密码回车
        
        # 用scp将备份推送到从服务器指定目录，主从服务器安装scp
        > scp 备份名.sql root@从服务器ip:/放置目录
        ```
    * xtrabackup（对Innodb引擎，不阻塞操作；对非Innodb引擎，会阻塞）
    
        备份略
    
    从库导入数据：
    ```
    > mysql -uroot -p < 备份名.sql
    # 输入密码回车
    ```
    
5. 启动复制链路
    ```
    # 设置复制链路
    mysql> change master to master_host='主库主机ip', master_user='主库slave用户名', master_password='主库slave用户密码', master_log_file='监听的主库二进制日志文件名', master_log_pos=监听主库二进制日志文件位置;
    
    # 启动复制监听
    mysql> start slave;
    ```

基于日志点复制的优缺点：
    
* 优点：
    * MySQL最早支持的复制技术，bug较少
    * 对SQL查询没有任何限制
    * 故障处理较容易

* 缺点：
    * 故障转移时重新获取新主库的日志信息比较困难

#### 基于GTID复制

GTID即全局事务ID，其保证为每个在主上提交的事务在复制及集群中可以生成一个唯一的ID

GTID=source_id:transaction_id

步骤：
1. 在主服务器上建立复制账号（不要在从服务器建立相同账号）
    
    ```
    # 创建用户
    mysql> create user '用户名'@'ip段' indentified by '密码';
    
    # slave授权
    mysql> grant replication slave on *.* to '用户名'@'ip段';
    ```
2. 配置主数据库服务器

    ```
    # 启用bin_log日志
    bin_log=/usr/local/mysql/log/mysql-bin
    
    # 服务器ID 
    server_id=100
    
    # 启用GTID模式
    gtid_mode=on
    
    # 强制GTID一致性
    # 像create table … select和create temporary table语句，以及同时更新事务表和非事务表的SQL语句或事务都不允许执行
    enforce_gtid_consistency=on
    
    # 中继日志写入二进制文件（5.7前gdit模式必启动，5.7后可不用）
    log_slave_updates=on
    ```
3. 配置从数据库服务器

    ```
    # 服务器ID 
    server_id=101
    
    # 启用relay_log日志
    relay_log=/usr/local/mysql/log/relay_log
    
    # 启用GTID模式
    gtid_mode=on
    
    # 强制GTID一致性
    enforce_gtid_consistency=on
    
    # 中继日志写入二进制文件
    log_slave_updates=on
    
    # 非slave授权只读，建议
    read_only=on
    
    # 从服务器连接主服务器信息方式，建议
    master_info_repository=TABLE        # Innodb表
    
    # 中继日志存储方式，建议
    relay_log_info_repository=TABLE     # Innodb表
    ```
4. 初始化从服务器数据
 
    参考基于日志点的[初始化从服务器数据](#backup)

    主库备份记录的不是主库二级制文件及偏移量，而是最后的事务的GTID值
5. 启动基于GTID的复制

    ```
    # 设置复制链路
    mysql> change master to master_host='主库主机ip', master_user='主库slave用户名', master_password='主库slave用户密码', master_auto_position=1;
    
    # 启动复制监听
    mysql> start slave;
    ```

基于GTID复制的优缺点：
    
* 优点：
    * 可以很方便的进行故障转移
    *从库不会丢失主库上的任何修改

* 缺点：
    * 故障处理比较复杂
    * 对执行的SQL有一定的限制

#### 选择复制模式
* 所使用的MySQL版本（GTID 5.6+）
* 复制架构及主从切换的方式（GTID较方便）
* 所使用的高可用管理组件
* 对应用的支持程度

#### MySQL复制拓扑

MySQL5.7+，支持一丛多主架构

![主从架构](./img/05.1.png?raw=true "主从架构")

一主多从拓扑：

![一主多从拓扑](./img/05.2.png?raw=true "一主多从拓扑")

* 优点：
    * 配置简单
    * 可以用多个从库分担读负载
* 用途：
    * 为不同业务使用不同的从库
    * 将一台从库放到远程IDC，用做灾备恢复
    * 分担主库的读负载

主-主复制拓扑：

![主-主复制拓扑](./img/05.3.png?raw=true "主-主复制拓扑")

* 主主模式

    产生数据冲突而造成复制链路中断，耗费大量时间，造成数据丢失
    
    * 两个主所操作的表最好能够分开
    * 使用下面两个参数控制自增ID的生成
    
        ```
        auto_increment_increment=2    # 自增ID步长
        auto_increment_offset=1|2     # 自增ID初值
        ```
* 主备模式（高可用）

    只有一台主服务器对外提供服务，另一台处于只读状态并只作为热备使用
    
    对外服务器故障或计划性维护时才会切换，备库升级主库，主库降为备库或下线维护再上线
    
    * 确保两台服务器上的初始数据相同
    * 确保两台服务器已经启动binlog并由不同的server_id
    * 两台服务器上启用log_slave_updates参数
    * 初始备库上启用read_only

主主-主从拓扑

![主主-主从拓扑](./img/05.4.png?raw=true "主主-主从拓扑")

级联复制：

![级联复制](./img/05.5.png?raw=true "级联复制")

#### MySQL复制性能优化

![主从复制](./img/05.6.png?raw=true "主从复制")

影响主从延迟的因素：
* 主库写入二进制日志的时间
    * 缓解：可以控制主库的事务大小，分割大事务
* 二进制日志传输时间
    * 缓解：使用MIXED日志格式或行格式（set binlog_row_image=minimal;）
* 默认只有一个SQL线程，主上并发的修改在从上变成了串行
    * 解决：使用多线程复制（5.6+），5.7中可按照逻辑时钟的方式分配SQL线程
    
        ```
        # 停止链路复制
        mysql> stop slave;
        
        # 设置多线程为逻辑时钟的方式（默认DATABASE）
        mysql> set global slave_paraller_type='logical_clock';
        
        # 设置4个线程
        mysql> set global slave_parallel_workers=4;
        
        # 启动链路复制
        mysql> start slave;
        ```

#### MySQL复制常见问题

数据损坏或丢失引起的主从复制错误：
* 主库或从库意外宕机

    处理：
    1. 跳过二进制日志事件或注入空事务的方式先恢复中断的复制链路
    2. 使用其他方法对比主从服务器上的数据，从新同步丢失数据

* 主库上二进制日志损坏

* 备库中继日志损坏
    * 若主库二进制日志正常，可从损坏位置重新同步

* 从库上进行数据修改造成主从复制错误
    * 从库设置read_only 
    * 主库数据和从库数据间取舍

* 不唯一的server_id或server_uuid

* 从服务器max_allow_packet设置引起的主从复制错误

#### MySQL复制无法解决的问题

* 分担主库写负载
    * 可分库分表
* 自动进行故障转移及主从切换
* 提供读写分离功能

#### 高可用架构

高可用性H.A.（high Availability）指的是通过尽量缩短因日常维护操作（计划）和突发的系统泵空（非计划）所导致的停机事件，以提高系统和应用的可用性

N个9可用性 => (365*24*60) * 1E-N  分钟 / 每年

5个9可用性 = 365*24*60 * 0.00001 = 5.256分钟/年

如何实现高可用：
* 避免导致系统不可用的因素，减少系统不可用的时间
    * 服务器磁盘空间耗尽
    
        备份或查询日志突增导致磁盘空间占满，MySQL无法记录二进制日志，无法记录请求而产生系统不可用的故障
        * 建立完善的监控及报警系统
        * 对备份数据进行恢复测试
        * 正确配置数据库环境（从库只读）
        * 对不需要数据进行归档和清理（不需要的Innodb数据归档到archive表）
    * 性能糟糕的SQL
    * 表结构和索引没有优化
    * 主从数据不一致
    * 人为的操作失误等等

* 增加系统冗余，保证发生系统不可用时可以尽快恢复
    * 避免存在单点故障
        
        单点故障是指在一个系统中提供相同功能的组件只有一个，若组件失效，影响整个系统功能得正常使用

        组成应用系统得各个组件都可能成为单点
        * 利用SUN共享存储或DRDB磁盘复制解决，但都不是好的方案
        
            ![共享存储](./img/05.7.png?raw=true "共享存储")
        
            ![DRDB磁盘镜像](./img/05.8.png?raw=true "DRDB磁盘镜像")
        
        * 利用多写集群或NDB集群（节点主主，要求内存）解决
        
        * 利用MySQL主从复制解决
            
            解决主服务器单点
            
            * 主服务器切换，如何通知新的主服务器IP地址
            * 如何检查主服务器是否可用
            * 如何处理从服务器和新主服务器之间的复制关系
        
    * 主从切换及故障转移

#### MMM架构

Multi-Master REplication Manager 多主（主备-主从模式）复制管理器

![MMM架构](./img/05.9.png?raw=true "MMM架构")

监控和管理MySQL的主主复制拓扑，并在当前主服务器失效时，进行主和主备服务器之间的主从切换和故障转移等工作。

mmm 监控主从复制健康状况
* 主库出现宕机时进行故障转移并自动配置其他从对新主的复制
    * 如何找到从库对应的新的主库日志点的日志同步点
    * 如果存在多个从库出现数据不一致的情况如何处理
    * mmm对以上两点处理粗暴，繁忙系统可能丢失数据
* 对集群中每个服务器提供虚拟IP，1成写，多个读，写虚拟vip只能在两个主切换，读vip可以在所有主从服务器上切换

![MMM部署所需资源](./img/05.10.png?raw=true "MMM部署所需资源")

#### MMM部署步骤

![MMM演示拓扑图](./img/05.11.png?raw=true "MMM演示拓扑图")

1. 配置主主复制及主从同步集群

    主库备份并推送导入到其它库，设置主主复制及主从复制

    ```
    # 查看主从情况
    mysql> show master status\G
    mysql> show slave status\G
    ```
2. 安装主从节点所需要的支持包

    使用 yum 安装，自动补全依赖

3. 安装及配置mmm工具集  

    全节点操作：

    ```
    # 下载yam源
    > wget http://mirror.opencas.cn/epel/epel-release-latest-6.noarch.rpm
    > wget http://rpms.famillecollet.com/enterprise/remi-release-latest-6.rpm
    
    # 安装yam源
    > rpm -ivh eper-release-latest-6.noarch.rpm
    > rpm -ivh remi-release-latest-6.rpm
    
    # 修改配置：/etc/yum.repos.d/epel.repo
    # baseurl解除注释
    # mirrorlist注释
    
    # 修改配置：/etc/yum.repos.d/remi.repo
    enabled=1   # 0->1
    
    # 查询支持的mmm包
    > yum search mmm
    
    # 安装mmm包
    > yum install mysql-mmm-agent.noarch -y
    ```  
    
    监控节点安装监控所需包：`> yum -y install mysql-mmm*`
    
    主库mmm相关用户创建：
    
    ```
    # 建立监控用户
    mysql> grant replication client on *.* to 'mmm_monitor'@'192.168.3.%' indentified by '123456';
    
    # 建立代理用户
    mysql> grant super,replication,client,process client on *.* to 'mmm_agent'@'192.168.3.%' indentified by '123456';
    
    # 复制用户即slave用户已建立
    ```

    主从节点mmm配置：
    
    1. /etc/mysql-mmm/mmm_common.conf
    
        ![mmm_comment配置](./img/05.12.png?raw=true "mmm_comment配置")  
        ![mmm_comment配置](./img/05.13.png?raw=true "mmm_comment配置")  
        ![mmm_comment配置](./img/05.14.png?raw=true "mmm_comment配置")
    
    2. /etc/mysql-mmm/mmm_agent.conf
    
        ```
        this db1 # 根据节点实际情况设置
        ```
    
    监控节点配置（这里是db3，即从库）：
    
    * /etc/mysql-mmm/mmm_mon.conf
    
        ![mmm_mon配置](./img/05.15.png?raw=true "mmm_mon配置")
        ![mmm_mon配置](./img/05.16.png?raw=true "mmm_mon配置")
    
4. 运行mmm监控服务

    启动主从节点mmm代理：
    
    ```
    > /etc/init.d/mysql-mmm-agent start
    ```
    
    启动监控节点（这是db3）监控：
    
    ```
    > /etc/init.d/mysql-mmm-monitor start
    ```
    
    监控节点查看状态：
    
    ```
    > mmm_control show
    # 看看几个节点ip及在线状态
    ```
    
    主从节点上查看网络：
    
    ```
    > ip addr
    # 看看是否有自身读写虚拟ip
    ```
    
5. 测试

    关闭主服务器MySQL：`> /etc/init.d/mysqld stop`
    
    监控节点查看状态：
    
    ```
    > mmm_control show
    ```
    
    主从节点上查看网络：
    
    ```
    > ip addr
    ```
    
    从节点查看主节点信息是否变化：
    
    ```
    mysql> show slave status \G
    ```
    
#### MMM集群优缺点

优点：
* 使用Perl脚本开发，完全开源
* 提供了读写vip（虚拟ip），是服务器角色的变更对前端应用透明
* 提供了从服务器的延迟监控
* 提供了主库故障转移后从库对新主的重新同步功能
* 很容易对发生故障的主数据库重新上线

缺点：
* 发布时间比较早，不支持MySQL新的复制功能（比如不支持GTID，不支持多线程）
* 没有读负载均衡的功能
* 进行主从切换时，容易造成数据丢失
* 监控服务存在单点故障

#### MHA架构

MHA（Master High Avaliability），由Perl脚本开发，支持基于日志点和基于GTID的复制，推荐GTID

![MHA](./img/05.17.png?raw=true "MHA")

功能：
* 监控主数据库服务器是否可用
* 主库不可用时，从多个从服务器选举出新主数据库服务器
* 提供了主从切换和故障转移功能
* MHA可以与5.5+的半同步分支结合

MHA是如何进行主从切换的：
* 尝试从故障的主数据库保存二进制日志
* 从多个备选（可认为设置一些服务器不参与选举）从服务器选举出新的备选主服务器
* 在备选主服务器和其它从服务器之间同步差异二进制数据
* 应用从原主服务器上保存的二进制日志（前提是获取保存成功）
* 提升备选主服务器作为新的主服务器
* 迁移集群中的其它从库作为新主的从服务器

![MHA 演示架构](./img/05.18.png?raw=true "MHA 演示架构")

#### MHA配置步骤
1. 集群内所有主机的SSH免认证登录

    ```
    # 生成ssh密钥
    > ssh-keygen
    # 回车直至完成
    
    # 推送密钥到集群中其它主机
    > ssh-copy-id -i /root/.ssh/id_rsa '-p 22 root@ip地址'
    # 输入推送目标root密码回车
    ```
    
2. 集群节点安装MHA-node软件包，监控服务器安装MHA-manager软件包

    ![监控节点安装安装perl支持包](./img/05.19.png?raw=true "监控节点安装安装perl支持包")
    
    主从节点：
    
    ```
    # 安装perl支持包
    > yum -y install prel-DBD-MYSQL ncftp perl-DBI.x86
    
    # 下载并安装mha4mysql-node源
    > wget https://github.com/linyue515/mysql-master-ha/raw/master/mha4mysql-node-0.57-0.el7.noarch.rpm
    > rpm -ivh mha4mysql-node-0.57-0.el7.noarch.rpm
    ```
    
    监控节点：
    
    ```
    # 安装perl支持包
    > yum -y install perl-Config-Tiny.noarch perl-Time-HiRes.x86_64 perl-Parallel-Formanager perl-Log-Dispatch-Perl.noarch prel-DBD-MYSQL ncftp
    
    # 下载并安装mha4mysql-manager源
    > wget https://github.com/linyue515/mysql-master-ha/raw/master/mha4mysql-manager-0.57-0.el7.noarch.rpm
    > rpm -ivh mha4mysql-manager-0.57-0.el7.noarch.rpm
    ```
3. 建立主从复制集群

    使用GTID复制

4. 配置MHA管理软件

    监控服务器：
    
    ```
    # 创建mha工作目录
    > mkdir -p /home/mysql_mha
    
    # 创建mha目录
    > mkdir -p /etc/mha
    
    # 创建配置文件
    > vim /etc/mha/mha.cnf
    [server default]
    user=mha
    password=123456
    manager_workdir=/home/mysql_mha
    manager_log=/home/mysql_mha/manager.log
    remote_workdir=/home/mysql_mha
    ssh_user=root
    repl_user=repl
    repl_password=123456
    # 每秒联通主库检测
    ping_interval=1
    # 主库二进制日志目录：show variables like 'log_bin%'，集群中二进制目录最好配置一致
    master_bin_log_dir=/home/mysql/sql_log
    # 可选：主从切换后，把主虚拟ip绑定在新主上；虚拟vip漂移脚本
    master_id_failover_script=/user/bin/master_ip_failover
    # 默认情况下mha只通过manager检测主数据库是否可用，使用该脚本，更多途径检测；最好加上网关
    secondary_check_script=/user/bin/masterha_secondary_check -s 192.168.3.101 -s 192.168.3.102 -s 192.168.3.100
    [server1]
    hostname=192.168.3.100
    # 参与选举
    candidate_master=1
    [server2]
    hostname=192.168.3.101
    candidate_master=1
    [server3]
    hostname=192.168.3.102
    # 不参与选举
    no_master=1
    ```
    
    主从切换虚拟VIP漂移脚本（/user/bin/master_ip_failover）：
    
    ![VIP漂移脚本](./img/05.21.png?raw=true "VIP漂移脚本")
    ![VIP漂移脚本](./img/05.22.png?raw=true "VIP漂移脚本")
    ![VIP漂移脚本](./img/05.23.png?raw=true "VIP漂移脚本")
    ![VIP漂移脚本](./img/05.24.png?raw=true "VIP漂移脚本")
    
    主服务器：
    
    ```
    # 创建mha用户
    mysql> grant all privileges on *.* to mha@'192.168.3.%' identified by '123456';
    ```
    
    全节点：
    
    ```
    > mkdir -p /home/mysql_mha
    ```

5. 使用 masterha_check_ssh（检测ssh免认证） 和 masterha_check_repl（检测复制链路） 对配置进行检验

    监控服务器：

    ```
    # 检测ssh免认证
    > masterha_check_ssh --conf=/etc/mha/mysql_mha.cnf
    
    # 复制链路检测
    > masterha_check_repl --conf=/etc/mha/mysql_mha.cnf
    ```

6. 启动并测试MHA服务

    监控服务器：
    
    ```
    > nohup masterha_manager --conf=/etc/mha/mysql_mha.cnf &
    ```

    主服务器：
    
    ```
    # 指定虚拟vip
    > ifconfig eth0:1 192.168.3.90/24
    ```

7. 测试

    主服务器：
    
    ```
    # 停掉主服务器
    > /etc/init.d/mysqld stop
    
    # 查看vip，不见了
    > ip addr
    ```
    
    服务器2（应该升级为主）：
    
    ```
    # 查看vip，应该有
    > ip addr
    ```
    
    服务器3：
    
    ```
    # 查看复制状态，主库应该变为服务器2
    mysql> show slave status \G
    ```

#### MHA架构实力演示

![MHA架构实力演示](./img/05.20.png?raw=true "MHA架构实力演示")

#### MHA架构优缺点

优点：
* 同样是Perl语言开发的开源工具
* 可支持GTID的复制模式
* 在进行故障转移时，更不易产生数据丢失
* 同一个监控节点可以监控多个集群

缺点：
* 需要编写脚本或利用第三方工具实现vip配置
* MHA启动后只会对主库进行监控
* 需要基于SSH免认证配置，存在一定的安全隐患
* 没有提供从服务器的读负载均衡功能

#### 读写分离和负载均衡介绍

读写分离两种方式：
1. 程序实现
    * 优点：
        * 由开发人员控制，比较灵活
        * 由程序直接连接数据库，性能损耗比较少
    * 缺点：
        * 增加开发工作量，使程序代码更复杂
        * 认为控制，容易出错
2. 中间件

    mysql-proxy
    
    maxScale（免费、性能损耗小、读写分离、读负载均衡）
    
    * 优点：
        * 根据查询语法分析，自动完成读写分离
        * 对程序透明，对已有程序不用做调整
    * 缺点：
        * 增加中间层，查询效率有损耗（降低50%-70%）
        * 对延迟敏感业务无法自动在主库执行

负载均衡：
1. 程序轮询：增改从服务器需修改程序
2. 软件/硬件
    软件：LVS/Haproxy/MaxScale
    硬件：F5

使用中间件，要进行基准测试

#### MaxScale实例演示

MariaDB公司发布的支持高可用的、负载均衡的就是有有良好扩展性的插件式中间件

![MaxScale的插件](./img/05.25.png?raw=true "MaxScale的插件")

安装：

![MaxScale安装](./img/05.26.png?raw=true "MaxScale安装")

```
# 下载maxscale源
> wget https://downloads.mariadb.com/enterprise/t6c0-wrnh/mariadb-maxscale/1.3.0/rhel/7/x86_64/maxscale-1.3.0-1.rhel7.x86_64.rpm

# 安装依赖包
> yum install libaio.x86_64 libaio-devel.x86_64 novacom-server.x86_64 -y

# 安装maxscale
> rpm -ivh maxscale-1.3.0-1.rhel7.x86_64.rpm
```

主数据库创建监控用户和路由用户：

```
# 建立监控用户 
mysql> create user scalemon@'192.168.3.%' identified by '123456';
mysql> grant replication slave,replication client on *.* to scalemon@'192.168.3.%';

# 建立路由模块用户 
mysql> create user maxscale@'192.168.3.%' identified by '123456';
mysql> grant select on mysql.* to maxscale@'192.168.3.%';
```

监控节点：

```
# 生成加密器：/var/lib/maxscale/
> maxkeys

# 加密123456 
> maxpasswd /var/lib/maxscale/ 123456

# 编辑/etc/maxscale.cnf
threads=4   # 线程

[server1]
type=server
address=192.168.3.100
port=3306
protocol=MySQLBackend
[server2]
type=server
address=192.168.3.101
port=3306
protocol=MySQLBackend
[server3]
type=server
address=192.168.3.102
port=3306
protocol=MySQLBackend

[MySQL Monitor]
type=monitor
module=mysqlmon
servers=server1,server2,server3
user=scalemon
passwd=123456
monitor_interval=1000  # ms

# 删除 [Read-Only Service] [Read-Only Listener] 模块

[Read-Write Service]
type=service
router=readwritesplit
servers=server1,server2,server3
user=maxscale
passwd=123456
max_slave_connections=100%      # 具体个数或百分比
max_slave_replication_lag=60    # 从服务器最大延迟（s），超过则不参与读写分离

[Read-Write Listener]
type=listener
service=Read-Write Service
protocol=MySQLClient
port=4006   # 推荐使用3306，此案例监听服武器同时是从库
```

监控服务器：

```
# 启动maxscale
> maxscale --config=/etc/maxscale.cnf

# 登录管理
> maxadmin --user=admin --password=mariadb

# 查看服务器
MaxScale> list servers

# 查看读写分离用户
MaxScale> show dbusers "Read-Write Service"
```

![最终高可用架构](./img/05.27.png?raw=true "最终高可用架构")
