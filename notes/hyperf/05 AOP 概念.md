#### phpstorm 配置 PHP 脚本
文件路径说明栏右侧：Add Configuration -> + -> PHP Script  

填入 Name（名称）、File（脚本文件）、Arguments（参数）

点击右侧启动/重启按钮

#### Hyperf 启动打印日志类型
config/config.php 文件 StdoutLoggerInterface::class -> log_level 项

### AOP 是什么
* 面向切面编程，通过 预编译 方式或 运行期动态代理 实现程序功能的统一维护的一种技术
* AOP 和 OOP 的设计思想有本质差异
* OOP 是针对业务处理过程的实体及其属性和行为进行抽象封装，以获得更加清晰高效的逻辑单元划分
* 而 AOP 则是针对业务处理过程中的切面（模块之间、方法之间、方法前后）进行提取，它所面对的是处理过程中的某个步骤或阶段，以获得逻辑过程中各部分之间低耦合性的隔离效果

### Hyperf 里的 AOP
* 只需理解一个理念，即 Aspect（切面）
* Aspect 可以切入任意类或任意注解类
* 被切入的类必须由 DI 管理
* 相对其它框架的 AOP 实现，这里做了简化，仅存在环绕（Around）一种通用形式，不做细的划分（如 Before,After,AfterReturning,AfterThrowing）。
* 基于 DI 实现，必须使用 hyperf/di 组件
* 通过 DI 创建的类才能使 AOP 生效，直接 new 不行
* 当代理类缓存文件（runtime/container/proxy目录下）不存在时才会重新生成代理类（2.0重启服务器会更新）

### AOP 的应用场景
* 参数校验、日志、无侵入埋点、安全控制、性能统计、事务处理、异常处理、缓存、无侵入监控、资源池、连接池管理等

### 基于 AOP 的功能
* @Cacheable、@CachePut、@CacheEvict
* @Task
* @RateLimit
* @CircuitBreaker
* 调用链追踪的无侵入埋点

### 示例
1. app/Controller/IndexController.php
    
    ```php
    <?php
    
    declare(strict_types=1);
    
    namespace App\Controller;
    
    use App\Annotation\Foo;
    use Hyperf\HttpServer\Annotation\AutoController;
    
    /**
     * @AutoController()
     * @Foo(bar="123", dar="321")
     */
    class IndexController extends AbstractController
    {
        public function index()
        {
            return PHP_EOL . 'xxxxx' . PHP_EOL;
        }
    }
    ```
2. app/Aspect/IndexAspect.php

    ```php
    <?php
    
    declare(strict_types=1);
    
    namespace App\Aspect;
    
    use App\Annotation\Foo;
    use App\Controller\IndexController;
    use Hyperf\Di\Annotation\Aspect;
    use Hyperf\Di\Aop\AbstractAspect;
    use Hyperf\Di\Aop\ProceedingJoinPoint;
    
    /**
     * @Aspect()
     */
    class IndexAspect extends AbstractAspect
    {
        // 要切入的类、类某个或某些（通过 * 模糊匹配）方法
        public $classes = [
            IndexController::class . '::index'
        ];
    
        # 要切入的注解，具体切入的还是使用了这些注解的类，仅可切入类注解和类方法注解
        public $annotations = [
    
        ];
    
        public function process(ProceedingJoinPoint $proceedingJoinPoint)
        {
            // 调用前进行某些处理
            $result = $proceedingJoinPoint->process();
            // 调用后进行某些处理
    
            /** @var Foo $foo */
            $foo = $proceedingJoinPoint->getAnnotationMetadata()->class[Foo::class];
            $bar = $foo->bar;
            
            return $bar . 'before' . $result . 'after';
        }
    }
    ```
