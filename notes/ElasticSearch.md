## 全文检索
是指计算机索引程序扫描文章，对每个关键词建立一个索引，指明该字词在文章中出现的次数和位置，当用户查询时，检索程序就根据事先建立的索引进行查找，并将结果反馈给用户的检索方式。

类似通过字典目录查询某些字词的过程。

## 全文检索架构
1. Lucene（apache 基金会的一个顶级项目）  

    官网：https://lucene.apache.org/  
    
    Lucene 是一个开源的搜索架构
    * Lucene core：基于 java 开发，包含搜索库、搜索索引、关键字的高亮显示、分词器...  
    * Solr：基于 Lucene core 开发，提供了 json、py、ruby、java 的各种 API  
    * PyLucene：支持 python
    
2. ElasticSearch

    ES 是构建于 Lucene 上的一个分布式 Restful 风格的数据搜索引擎，数据存储只支持 json 格式
    
    三大核心：Index（相当于库）、Type（相当于表）、Document（相当于表数据）
    
    中文社区：https://elasticsearch.cn/
    
    全文指南：https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html
    
    PHP 操作：https://www.elastic.co/guide/cn/elasticsearch/php/current/index.html

3. Solr

    Solr 是构建于 Lucene 上的搜索框架，支持分布式，支持 json、xml、csv 等格式

## ES 和 Solr 比较
1. 配置

    ES 配置简单，下载后直接可以用，使用 IK 分词器也很简单

    Solr 配置比较复杂

2. 分布式

    ES 自带分布式协调功能，分布式需要服务器更少

    Solr 需使用 zookeeper 进行分布式管理，分布式需要服务器更多

3. 数据格式

    ES 只支持 json

    Solr 功能多，自带可视化界面

4. 功能

    ES 注重核心功能，拓展高级特性需第三方插件，可视化借助于 kibana

    Solr 功能多，自带可视化界面

5. 性能

    ES 建立索引非常快，可用于实时性查询，适用于新型实时搜索应用，如 facebook

    Solr 查询效率块，但索引更新特别慢，适用于大型传统查询，如淘宝（项目运行前维护数据，不需要高效；查询需要高效）

6. 社区

    ES 相对用户量少，版本更新太快，学习成本更高

    Solr 比较成熟，用户量大，社区厉害，问题容易解决

## ES 和 Lucene 比较
* Lucene 是一个搜索架构，提供了索引和全文检索，需要开发人员根据公司业务自己实现搜索模块  
  适用于业务繁琐、市面上搜索引擎不能满足业务需求的公司

* ES 是基于 Lucene 开发的框架，实现了较为完善的搜索模块，可直接使用

## ES 在 linux 的配置
1. 修改主机名，明示主机上搭建的的主要 软件/架构/中间件 等  

    ```
    # /etc/sysconfig/network（修改）
    HOSTNAME=ES服务名 # 如 ES01
    ```
2. 修改 HOSTNAME 和 IP 映射，便于其他服务器 ssh 访问本机

    ```
    # /etc/hosts（追加）
    192.168.xxx.xxx 主机名
    ```
3. 防火墙设置（完成后重启服务器：`# reboot`）
    * 内网使用，可关闭防火墙及其开机启动
    
        ```
        # 关闭防火墙
        $ service iptables stop
        # 关闭防火墙开机自启
        $ chkconfig iptables off
        ```
    * 外网使用，开放端口
4. 配置 jdk(jdk8)
    1. 获取 tar.gz 包并解压：`# tar -zxvf 包名`
    2. 配置 jdk 变量

        ```
        # /etc/profile（追加）
        export JAVA_HOME=jdk目录绝对路径
        export PATH=$PATH:$JAVA_HOME/bin
        ```
        使修改立即生效：`# source /etc/profile`
    3. 检测配置是否成功：`# javac -version`    
5. 配置 ES 
    1. 获取 tar.gz 包并解压：`# tar -zxvf 包名`

    2. <a id="config2">增大 linux 部署软件能打开的内存和硬盘的文件数</a>
    
        ```
        # /etc/security/limits.conf （追加）
        * soft nproc 655350
        * hard nproc 655350
        * soft nofile 655350
        * hard nofile 655350
        ```

    3. <a id="config3">开启最大线程数</a>（系统保护机制，默认不会开大）
    
        ```
        # /etc/sysctl.conf（追加）
        vm.max_map_count = 262144
        ```

    4. <a id="config4">设置用户最大线程数</a>
    
        ```
        # /etc/security/limits.d/90-nproc.conf（修改）
        * soft nproc 4096 # 默认1024
        ```

    5. 使配置永久生效：`# sysctl -p`

    6. <a id="config6">修改 ES 配置文件</a>
    
        ```
        # ./config/elasticsearch.yml（修改）
          
        # 集群名称（随意）
        cluster.name: my-cluster
        # 节点名称，单节点随意，集群不能重复
        node.name: node-1
          
        # 节点数据存放目录，建议 ES 下 data（无则创建）目录
        path.data: ES绝对路径/data
        # 节点日志目录
        path.logs: ES绝对路径/logs
          
        # 放开 ES 内存锁，让 ES 拥有最大内存权限
        bootstrap.memory_lock: false
        # 关闭 seccomp 检测
        bootstrap.system_call_filter: false
          
        # 默认本机访问，指定访问 IP，不限制则 0.0.0.0
        netwrok.host: 192.168.xxx.xxx
          
        # 端口号
        http.port: 9200
        
        # 集群中节点 IP 列表
        discovery.zen.ping.unicast.hosts: ["192.168.xxx.xxx"]
        ```

    7. <a id="config7">设置 ES 用户</a>（不能用 root 启动）
    
        ```
        # 创建 ES 用户，这里以 esuser 用户为例
        $ useradd esuser
        # 设置密码
        $ passwd esuser
        # 修改 ES 用户
        $ chown -R esuser ES目录路径
        # 切换 esuser 用户
        $ su esuser
        ```

    8. 启动：`$ ./bin/elasticsearch`

    9. 验证，浏览器 IP:PORT 访问

    10. 常见问题：

        1. 禁用 root 启动，参考 [设置 ES 用户](#config7) 处理
        
        2. seccomp unavailable 错误，两个解决方法：
            * 升级系统到 CentOS7 或更高版本
            * 参考 [修改 ES 配置文件](#config6)，关闭 seccomp 检测
            
        3. ES 能打开的文件太少，根据提示，参考 [增大 linux 部署软件能打开的内存和硬盘的文件数](#config2) 处理
        
        4. ES 能用的线程数太少，根据提示，参考 [设置用户最大线程数](#config4) 处理
        
        5. 内存区域太少，根据提示，参考 [开启最大线程数](#config3) 处理
        
        6. 首次用 root 启动，创建了 elasticsearch.keystore 文件，其他用户无权操作，将该文件所属改为 ES 指定用户

## 配置中文分词器 IK（国人开发）
1. ES 安装：`$ ./bin/elasticsearch-plugin install 同版本IK包路径（可以是网络地址）`

2. 普通安装
    1. 在 ES 下的 plugins 目录中创建 IK 目录
    
    2. 获取 ES 同版本 IK 分词器 zip 包
    
    3. 将 zip 放于 IK 目录并解压：`# unzip 包名`

## kibana（开源的数据分析和可视化平台，主要用于 ES）
1. 配置
    1. 获取与 ES 同版本的 tar.gz 包并解压：`# tar -zxvf 包名`

    2. 配置 jdk 环境变量（参见 ES 不需另外配置）

    3. 修改配置文件
        ```
        # ./config/kibana.yml
        server.port: 5601
        server.host: "192.168.xxx.xxx"
        elasticsearch.url: "http://192.168.xxx.xxx:9200"
        ```

    4. 软件启动：`# ./bin/kibana`

    5. 三种状态：绿色正常、黄色警告、红色报错

2. 浏览器 IP:PORT 访问

3. 操作（Dev Tools -> console）
    1. 查询所有

        ```
        GET _search
        {
            "query": {
                "match_all": {}
            }
        }
        ```

    2. 查看集群状态

        ```
        GET _cat/health
        GET _cat/health?v   # 字段说明
        ```

    3. 查看集群节点信息

        ```
        GET _cat/nodes
        GET _cat/nodes?v
        ```

    4. 查看所有 index 信息

        ```
        GET _cat/indices
        GET _cat/indices?v
        ```

    5. 创建 index

        ```
        PUT 不存在index名  
        POST 不存在index名
        ```

    6. 创建 type 

        ```
        PUT /index名
        {
            "mapping": {
                "不存在type名": {
                    "properties": {
                        "字段名": {"type": "字段类型"}
                    }
                }
            }
        }
        POST /不存在index名/不存在type名
        {
            "properties": {
                "字段名": {"type": "字段类型"}
            }
        }
        ```

    7. 查看 type

        ```
        GET /index名/_mapping/type名
        ```

    8. 向 type 添加数据

        ```
        PUT /index名/type名/指定ID
        {
            "字段名": 值
        }
        POST /index名/type名
        {
            "字段名": 值
        }
        ```

    9. 查看数目

        ```
        GET /index名/type名/ID
        GET /index名/type名/ID/_source
        ```

    9. 删除数据

        ```
        DELETE /index名/type名/ID
        ```
