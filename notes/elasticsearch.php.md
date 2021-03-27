### 官方文档
https://www.elastic.co/guide/cn/index.html

### composer 安装
* 修改 composer.json 文件

    ```
    {
        "require": {
            "elasticsearch/elasticsearch": "~6.0"
        }
    }
    ```
* composer 命令：`composer require elasticsearch/elasticsearch:~6.0`
### 客户端生成器
```php
# composer 自加载
require 'vendor/autoload.php';
  
# 创建 ElasticSearch 客户端生成器
$clientBuilder = Elasticsearch\ClientBuilder::create();
```
### 设置主机（[scheme://][user:pass@]host[:port]）
```php
# 默认主机配置
$hosts = ['localhost:9200'];

# 一维数组
$hosts = [
    '192.168.1.1',
    'mydomain.server.com:9201',
    'https://192.168.1.2:9202',
    'https://user:password@localhost:9203',
];

# 二维数组
$hosts =  [
    [
        'host' => 'localhost',
        'port' => '9203',
        'scheme' => 'https',
        'user' => 'user',
        'pass' => 'password'
    ],
    [
        'host' => 'localhost',
    ]
];

$clientBuilder->setHosts($hosts);     
```
### SSL 加密
+ 公共 CA 证书过期（安装 composer/ca-bundle）

    ```php
    $caBundle = \Composer\CaBundle\CaBundle::getBundledCaBundlePath();
    $clientBuilder->setSSLVerification($caBundle);
    ```
+ 自签名证书
    ```php
    $myCert = 'path/to/cacert.pem';
    $clientBuilder->setSSLVerification($myCert);
    ```
    
### 设置重连次数
```php
# 连接拒绝超时重连次数限制，默认节点数
$clientBuilder->setRetries(2);
```
### 开启日志（安装 monolog/monolog:~1.0）
```php
# 5.x
$logger = Elasticsearch\ClientBuilder::defaultLogger('日志文件路径');
# 6.x
$logger = new Monolog\Logger('名称');
$logger->pushHandler(new Monolog\Handler\StreamHandler('日志文件路径', Monolog\Logger::WARNING));

$clientBuilder->setLogger($logger);
```
### 设置 HTTP handler
```php
use Elasticsearch\ClientBuilder;

$handler = ClientBuilder::defaultHandler(); # 默认，异步模式开启会转为 multiHandler
$handler  = ClientBuilder::singleHandler();
$handler   = ClientBuilder::multiHandler();
$handler  = new 自定义handler();

$clientBuilder->setHandler($handler);
```

### 设置连接池
```php
$pool = new Elasticsearch\ConnectionPool\StaticNoPingConnectionPool(); # 默认
$pool = new Elasticsearch\ConnectionPool\SimpleConnectionPool();
$pool = new Elasticsearch\ConnectionPool\SniffingConnectionPool();
$pool = new Elasticsearch\ConnectionPool\StaticConnectionPool();
$pool = new Elasticsearch\ConnectionPool\StaticNoPingConnectionPool();
$pool  = new 自定义pool();

$clientBuilder->setConnectionPool($pool);
```
### 设置选择器
```php
$selector = new Elasticsearch\ConnectionPool\Selectors\RoundRobinSelector(); # 默认
$selector = new Elasticsearch\ConnectionPool\Selectors\RandomSelector();
$selector = new Elasticsearch\ConnectionPool\Selectors\StickyRoundRobinSelector();
$selector  = new 自定义selector();

$clientBuilder->setSelector($selector);
```
### 设置序列化器
```php
$serializer = new Elasticsearch\Serializers\SmartSerializer(); # 默认
$serializer = new Elasticsearch\Serializers\ArrayToJSONSerializer();
$serializer = new Elasticsearch\Serializers\EverythingToJSONSerializer();
$serializer  = new 自定义serializer();

$clientBuilder->setSerializer($serializer);
```
### 设置 connectionFactory
```php
$connectionFactory = new Elasticsearch\Connections\ConnectionFactory(); # 默认
$connectionFactory  = new 自定义connectionFactory();

$clientBuilder->setConnectionFactory($connectionFactory);
```

### 设置 Endpoint 闭包
```php
# 默认闭包
$serializer = new Elasticsearch\Serializers\SmartSerializer();
$newEndpoint = function ($class) use ($serializer) {
    $fullPath = '\\Elasticsearch\\Endpoints\\' . $class;
    if ($class === 'Bulk' || $class === 'Msearch' || $class === 'MsearchTemplate' || $class === 'MPercolate') {
        return new $fullPath($serializer);
    } else {
        return new $fullPath();
    }
};

$clientBuilder->setEndpoint($newEndpoint);
```

### 设置序列化器
```php
$connectionFactory = new Elasticsearch\Connections\ConnectionFactory(); # 默认
$connectionFactory  = new 自定义connectionFactory();

$clientBuilder->setConnectionFactory($connectionFactory);
```

### 设置序列化器
```php
$connectionFactory = new Elasticsearch\Connections\ConnectionFactory(); # 默认
$connectionFactory  = new 自定义connectionFactory();

$clientBuilder->setConnectionFactory($connectionFactory);
```

### 客户端对象
```php
# 生成客户端对象
$client = $clientBuilder->build();
```

### 从 hash 配置中创建对象
```php
# 配置参数组，key 等效于 $clientBuilder->setKey(value)
$params = [
    'hosts' => [
        'localhost:9200'
    ],
    'retries' => 2,
    'handler' => \Elasticsearch\ClientBuilder::singleHandler()
];

# 未知参数是否抛出异常，默认不抛出
$quiet = false;

$client = \Elasticsearch\ClientBuilder::fromConfig($params, $quiet);
```
### 重连异常
```php
try {
    // 模拟操作
} catch (Elasticsearch\Common\Exceptions\TransportException $e) { # 连接异常
    $previous = $e->getPrevious();
    if ($previous instanceof Elasticsearch\Common\Exceptions\MaxRetriesException) {
        echo "重连次数达到上限！";
    } elseif ($previous instanceof \Elasticsearch\Common\Exceptions\Curl\OperationTimeoutException) {
        echo "连接超时";
    } elseif ($previous instanceof \Elasticsearch\Common\Exceptions\Curl\CouldNotConnectToHost) {
        echo "连接拒绝";
    } elseif ($previous instanceof \Elasticsearch\Common\Exceptions\Curl\CouldNotResolveHostException) {
        echo "DNS 查找超时";
    }
}
```

### 按请求配置
* 忽略异常

    ```php
    $params = [
        'index'  => 'test_missing',
        'type'   => 'test',
        'id'     => 1,
    ];
    
    # 忽略 404 异常
    $params['client']['ignore'] = 404;
    
    # 忽略 404,405 异常
    $params['client']['ignore'] = [404, 405];
    
    # 获取文档
    $client->get($params);
    ```
* 自定义查询参数

    ```php
    $params = [
        'index' => 'test',
        'type' => 'test',
        'id' => 1,
        'parent' => 'abc',              // white-listed Elasticsearch parameter
        'client' => [
            'custom' => [
                'customToken' => 'abc', // user-defined, not white listed, not checked
                'otherToken' => 123
            ]
        ]
    ];
    $exists = $client->exists($params);
    ```
* 返回详细输出

    ```php
    $params = [
        'index' => 'test',
        'type' => 'test',
        'id' => 1
    ];
    
    # 详细输出
    $params['client']['verbose'] = true;
    
    $response = $client->get($params);
    print_r($response);
    ```
* Curl 超时设置（超时服务端会）

    客户端超时不意味服务端终止请求，服务端会继续直到请求完成。
    
    慢查询或 bulk 请求下，操作在后台继续执行，客户端超时未处理服务器回压（背压），可能造成服务端过载。

    ```php
    $params = [
        'index' => 'test',
        'type' => 'test',
        'id' => 1,
        'client' => [
            'timeout' => 10,        // 请求上限 10 s
            'connect_timeout' => 10 // 连接上限 10 s
        ]
    ];
    $response = $client->get($params);
    ```
* 开启 Future（异步） 模式
    ```php
    $futures = [];
    
    for ($i = 0; $i < 1000; $i++) {
        $params = [
            'index' => 'test',
            'type' => 'test',
            'id' => $i,
            'client' => [
                'future' => 'lazy'
            ]
        ];
    
        // 请求会加入队列，默认每批100个，批解析（curl 批处理）
        $futures[] = $client->get($params);
    }
    
    
    foreach ($futures as $future) {
        // 返回结果，若无结果，则强制解析（同批次一同解析）
        echo $future['_source'];
        # 同上
        # print_r($future->wait());
    }
    
    ```
  
    更改批量值
    ```php
    $defaultHandler = Elasticsearch\ClientBuilder::defaultHandler(['max_handles' => 500]); # 默认100
    $clientBuilder->setHandler($defaultHandler);
    ```
* SSL 加密

    ```php
    $params = [
        'index' => 'test',
        'type' => 'test',
        'id' => 1,
        'client' => [
            'verify' => 'path/to/cacert.pem'      // Use a self-signed certificate
        ]
    ];
    $result = $client->get($params);
    ```
### PHP 处理 JSON 数组或对象
* 空对象

    ```php
    $params['body'] = [
        'query' => [
            'match' => [
                'content' => 'quick brown fox'
            ]
        ],
        'highlight' => [
            'fields' => [
                # "content": {}
                'content' => new \stdClass()  # 这里不能为空数组，会 json_encode 成 []
            ]
        ]
    ];
    $results = $client->search($params);
    ```
* 对象数组

    ```php
    $params['body'] = [
        'query' => [
            'match' => [
                'content' => 'quick brown fox'
            ]
        ],
        # "sort": []
        'sort' => [ 
            ['time' => ['order' => 'desc']], 
            ['popularity' => ['order' => 'desc']]
        ]
    ];
    $results = $client->search($params);
    ```
* 空对象数组

    ```php
    $params['body'] = [
        'query' => [
            'function_score' => [
                # "functions": []
                'functions' => [ 
                    [ 
                        # "random_score": {}
                        'random_score' => new \stdClass() 
                    ]
                ]
            ]
        ]
    ];
    $results = $client->search($params);
    ```
### 索引管理
* 创建一个索引

    ```php
    $params = [
        'index' => 'my_index'
    ];
    $client->indices()->create($params);
    ```

    ```php
    $params = [
        'index' => 'my_index',
        'body' => [
            'settings' => [
                'number_of_shards' => 3,
                'number_of_replicas' => 2
            ],
            'mappings' => [
                'my_type' => [
                    '_source' => [
                        'enabled' => true
                    ],
                    'properties' => [
                        'first_name' => [
                            'type' => 'string',
                            'analyzer' => 'standard'
                        ],
                        'age' => [
                            'type' => 'integer'
                        ]
                    ]
                ]
            ]
        ]
    ];
    
    $client->indices()->create($params);
    ```

    ```php
    $params = [
        'index' => 'reuters',
        'body' => [
            'settings' => [
                'number_of_shards' => 1,
                'number_of_replicas' => 0,
                'analysis' => [
                    'filter' => [
                        'shingle' => [
                            'type' => 'shingle'
                        ]
                    ],
                    'char_filter' => [
                        'pre_negs' => [
                            'type' => 'pattern_replace',
                            'pattern' => '(\\w+)\\s+((?i:never|no|nothing|nowhere|noone|none|not|havent|hasnt|hadnt|cant|couldnt|shouldnt|wont|wouldnt|dont|doesnt|didnt|isnt|arent|aint))\\b',
                            'replacement' => '~$1 $2'
                        ],
                        'post_negs' => [
                            'type' => 'pattern_replace',
                            'pattern' => '\\b((?i:never|no|nothing|nowhere|noone|none|not|havent|hasnt|hadnt|cant|couldnt|shouldnt|wont|wouldnt|dont|doesnt|didnt|isnt|arent|aint))\\s+(\\w+)',
                            'replacement' => '$1 ~$2'
                        ]
                    ],
                    'analyzer' => [
                        'reuters' => [
                            'type' => 'custom',
                            'tokenizer' => 'standard',
                            'filter' => ['lowercase', 'stop', 'kstem']
                        ]
                    ]
                ]
            ],
            'mappings' => [
                '_default_' => [
                    'properties' => [
                        'title' => [
                            'type' => 'string',
                            'analyzer' => 'reuters',
                            'term_vector' => 'yes',
                            'copy_to' => 'combined'
                        ],
                        'body' => [
                            'type' => 'string',
                            'analyzer' => 'reuters',
                            'term_vector' => 'yes',
                            'copy_to' => 'combined'
                        ],
                        'combined' => [
                            'type' => 'string',
                            'analyzer' => 'reuters',
                            'term_vector' => 'yes'
                        ],
                        'topics' => [
                            'type' => 'string',
                            'index' => 'not_analyzed'
                        ],
                        'places' => [
                            'type' => 'string',
                            'index' => 'not_analyzed'
                        ]
                    ]
                ],
                'my_type' => [
                    'properties' => [
                        'my_field' => [
                            'type' => 'string'
                        ]
                    ]
                ]
            ]
        ]
    ];
    $client->indices()->create($params);
    ```
* 删除一个索引

    ```php
    $params = ['index' => 'my_index'];
    $client->indices()->delete($params);
    ```
* Put Settings API

    Put Settings API 允许你更改索引的配置参数:

    ```php
    $params = [
        'index' => 'my_index',
        'body' => [
            'settings' => [
                'number_of_replicas' => 0,
                'refresh_interval' => -1
            ]
        ]
    ];
    
    $client->indices()->putSettings($params);
    ```
* Get Settings API

    Get Settings API 可以让你知道一个或多个索引的当前配置参数：

    ```php
    // 获取单个索引
    $params = ['index' => 'my_index'];
    $client->indices()->getSettings($params);
    
    // 获取多个索引
    $params = [
        'index' => ['my_index', 'my_index2']
    ];
    $client->indices()->getSettings($params);
    ```
* Put Mappings API

    Put Mappings API 允许你更改或增加一个索引的映射。

    ```php
    // 设置索引和类型
    $params = [
        'index' => 'my_index',
        'type' => 'my_type2',
        'body' => [
            'my_type2' => [
                '_source' => [
                    'enabled' => true
                ],
                'properties' => [
                    'first_name' => [
                        'type' => 'string',
                        'analyzer' => 'standard'
                    ],
                    'age' => [
                        'type' => 'integer'
                    ]
                ]
            ]
        ]
    ];
    
    // 更新索引映射
    $client->indices()->putMapping($params);
    ```
* Get Mappings API

    Get Mappings API 返回索引和类型的映射细节。你可以指定一些索引和类型，取决于你希望检索什么映射。

    ```php
    // 获取所有索引和类型的映射
    $response = $client->indices()->getMapping();
    
    // 获取 my_index 中所有类型的映射
    $params = ['index' => 'my_index'];
    $response = $client->indices()->getMapping($params);
    
    // 获取所有类型的 my_type 的映射，而不考虑索引
    $params = ['type' => 'my_type'];
    $response = $client->indices()->getMapping($params);
    
    // 获取 my_index 中的 my_type 映射
    $params = [
        'index' => 'my_index',
        'type' => 'my_type'
    ];
    $response = $client->indices()->getMapping($params);
    
    // 获取多个索引的映射
    $params = [
        'index' => ['my_index', 'my_index2']
    ];
    $response = $client->indices()->getMapping($params);
    ```
* 索引命名空间下的其他 API

    Elasticsearch\Namespaces\IndicesNamespace 

### 索引文档
* 单一索引

    ```php
    $params = [
        'index' => 'my_index',
        'type' => 'my_type',
        'id' => 'my_id',  //  
        'routing' => 'company_xyz',
        'timestamp' => strtotime("-1d"),
        'body' => [ 'testField' => 'abc']
    ];
    
    
    $response = $client->index($params);
    ```






