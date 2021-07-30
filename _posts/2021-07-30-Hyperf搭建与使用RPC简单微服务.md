## Hyperf搭建与使用JSON RPC 简单微服务
### 背景：
因为项目需要，公司的要求进行系统重构。我们主营项目，不是一个单应用来的。产品想法好多，同一个项目光小程序端就有5个，APP5个。所以想寻求微服务架构方案。最近火了很久的Swoole一直没机会，现基于Hyperf的RPC简单微服务架构试探：


网上找了很多参考的技术帖子，但没有一个是可以让一个小白从头码到尾后可以成功运行的，都是讲得不清不楚或缺文件。还有官网也是，众所周知官方文档一般给熟手看的。在尝试着照着3个帖子敲代码无果后，我就放弃了。综合了几个技术文章，再细说阅读了官方手册，先理解了原理。然后再按照自己的理解去码代码，成功运行起来了。在此记录一下。

参考官方文档：[微服务](https://hyperf.wiki/2.2/#/zh-cn/microservice)

### 运行环境
Ubuntu 20.04（我用的是win10的WSL）

### 安装 Hyperf
执行下面的命令可以于当前所在位置创建一个 skeleton 项目
```
composer create-project hyperf/hyperf-skeleton
```
Hyperf 使用 Composer 来管理项目的依赖，在安装时，您可根据您自身的需求，对组件依赖进行选择。

### 本次用到的依赖组件
```
hyperf/json-rpc 
hyperf/rpc-client  
hyperf/rpc-server  
hyperf/service-governance 
hyperf/consul
```

### 使用
服务有两种角色，一种是 服务提供者(ServiceProvider)，即为其它服务提供服务的服务，另一种是服务消费者(ServiceConsumer)。

由于我只有一台机器，所以在同一台机器分别建2个目录来运行提供者与消费者。用不同的端口来区分

### 定义服务提供者
```
mkdir hyperf
# 安装服务端
composer create-project hyperf/hyperf-skeleton hyperf-provider
```
1、在app目录下新建JsonRpc先新建一个CalculatorServiceInterface接口文件，文件名CalculatorServiceInterface.php
```
<?php

declare(strict_types=1);

namespace App\JsonRpc;


interface CalculatorServiceInterface
{
    public function add(int $v1, int $v2): int;
}
```

2、基于这个Interface，在该目录下添加一个实现方式的CalculatorService，文件名CalculatorService.php
```
<?php

namespace App\JsonRpc;

use Hyperf\RpcServer\Annotation\RpcService;

/**
 * 注意，如希望通过服务中心来管理服务，需在注解内增加 publishTo 属性
 * @RpcService(name="CalculatorService", protocol="jsonrpc-http", server="jsonrpc-http")
 */
class CalculatorService implements CalculatorServiceInterface
{
    public function add(int $a, int $b): int
    {
        return $a + $b;
    }
}
```

3、定义 JSON RPC Server

HTTP Server (适配 jsonrpc-http 协议)
```
<?php

use Hyperf\Server\Server;
use Hyperf\Server\Event;

return [
    // 这里省略了该文件的其它配置
    'servers' => [
        [
            'name' => 'jsonrpc-http',
            'type' => Server::SERVER_HTTP,
            'host' => '0.0.0.0',
            'port' => 9502,
            'sock_type' => SWOOLE_SOCK_TCP,
            'callbacks' => [
                Event::ON_REQUEST => [\Hyperf\JsonRpc\HttpServer::class, 'onRequest'],
            ],
        ],
    ],
];

```
### 定义服务消费者
```
cd hyperf
# 安装消费端
composer create-project hyperf/hyperf-skeleton hyperf-consumer
```
1、同样在app目录下新建JsonRpc新建一个CalculatorServiceInterface接口文件，文件名CalculatorServiceInterface.php，同服务端保持一致。但在本项目并没有add 方法的实现，真正的add方法在服务提供者里已经实现了，hyperf会帮我们找到相应服务并使用。
```
<?php

declare(strict_types=1);

namespace App\JsonRpc;

interface CalculatorServiceInterface
{
    public function add(int $v1, int $v2): int;
}
```

2、在 config/autoload/services.php 配置文件内进行一些简单的配置，即可通过动态代理自动创建消费者类。如没此文件自行创建。
```
<?php
declare(strict_types=1);

use App\JsonRpc\CalculatorServiceInterface;

return [
    // 此处省略了其它同层级的配置
    'consumers' => [
        [
            // name 需与服务提供者的 name 属性相同
            'name' => 'CalculatorService',
            // 服务接口名，可选，默认值等于 name 配置的值，如果 name 直接定义为接口类则可忽略此行配置，如 name 为字符串则需要配置 service 对应到接口类
            'service' => CalculatorServiceInterface::class,
            // 对应容器对象 ID，可选，默认值等于 service 配置的值，用来定义依赖注入的 key
            'id' => \App\JsonRpc\CalculatorServiceInterface::class,
            // 服务提供者的服务协议，可选，默认值为 jsonrpc-http
            // 可选 jsonrpc-http jsonrpc jsonrpc-tcp-length-check
            'protocol' => 'jsonrpc-http',
            // 负载均衡算法，可选，默认值为 random
            'load_balancer' => 'random',
            // 这个消费者要从哪个服务中心获取节点信息，如不配置则不会从服务中心获取节点信息
            //'registry' => [
            //    'protocol' => 'consul',
            //    'address' => 'http://127.0.0.1:8500',
            //],
            // 如果没有指定上面的 registry 配置，即为直接对指定的节点进行消费，通过下面的 nodes 参数来配置服务提供者的节点信息
            // 注意这里端口写我在上面服务提供者定义的9602
            'nodes' => [
                ['host' => '127.0.0.1', 'port' => 9602],
            ],
            // 配置项，会影响到 Packer 和 Transporter
            'options' => [
                'connect_timeout' => 5.0,
                'recv_timeout' => 5.0,
                'settings' => [
                    // 根据协议不同，区分配置
                    'open_eof_split' => true,
                    'package_eof' => "\r\n",
                    // 'open_length_check' => true,
                    // 'package_length_type' => 'N',
                    // 'package_length_offset' => 0,
                    // 'package_body_offset' => 4,
                ],
                // 重试次数，默认值为 2，收包超时不进行重试。暂只支持 JsonRpcPoolTransporter
                'retry_count' => 2,
                // 重试间隔，毫秒
                'retry_interval' => 100,
                // 当使用 JsonRpcPoolTransporter 时会用到以下配置
                'pool' => [
                    'min_connections' => 1,
                    'max_connections' => 32,
                    'connect_timeout' => 10.0,
                    'wait_timeout' => 3.0,
                    'heartbeat' => -1,
                    'max_idle_time' => 60.0,
                ],
            ],
        ]
    ],
];
```

3、route.php路由修改添加一个测试入口
```
Router::addRoute(['GET', 'POST', 'HEAD'], '/add', 'App\Controller\IndexController@add');
```

4、这里为方便，直接修改\app\Controller\IndexController.php
```
<?php

declare(strict_types=1);
/**
 * This file is part of Hyperf.
 *
 * @link     https://www.hyperf.io
 * @document https://hyperf.wiki
 * @contact  group@hyperf.io
 * @license  https://github.com/hyperf/hyperf/blob/master/LICENSE
 */

namespace App\Controller;

use Hyperf\Utils\ApplicationContext;
use App\JsonRpc\CalculatorServiceInterface;

class IndexController extends AbstractController
{
    public function index()
    {
        $user = $this->request->input('user', 'Hyperf-consumer');
        $method = $this->request->getMethod();

        return [
            'method' => $method,
            'message' => "Hello {$user}.",
        ];
    }

    public function add()
    {
        // return 'add test~';
        $client = ApplicationContext::getContainer()->get(CalculatorServiceInterface::class);
        // var_dump($client);exit;
        $value = $client->add(10, 20);
        return $value;
    }
}

```

### 启动服务器
分别启动服务提供者与服务消费者
```
cd hyperf-provider
php bin/hyperf.php start

cd hyperf-consumer
php bin/hyperf.php start

```

### 测试
访问接口
```
http://127.0.01:9501/add
```

**注意：文档上说可以使用consul发布到服务中心；消费者使用注解注入了一个接口类。下次再实现。**