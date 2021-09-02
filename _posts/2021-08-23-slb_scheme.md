# SLB方案
/* @updated 2021-08-24 */

## 背景
我们在2021年初对XXX系统进行了一次架构升级演进：把原本MySQL主从模式1主N从，读写分离的数据库架构替换成了分布式数据库TiDB。
当解决了数据库的存储的性能瓶颈，新的瓶颈出现在了web容器的单体性能上。随着用户数的增长，并发压力主要落在应用服务的单体上，响应逐渐变慢。


## 目标
- 优化原有架构
- 用户状态、缓存等保存到专用的缓存服务器；附件保存到云存储中
- 集群部署和负载均衡
- 为日益增长的用户数，以及节假日前突发流量做准备


## 实现方案
系统目前还属于普通的Web开发，即常用的单应用模式。这种方式对于一般的Web应用，维护相对简单且使用很方便，完全能够胜任。但是对于高并发的企业级网站，就显得力不从心。单台服务器就算将性能优化得再好，也不能支撑大用户量的访问压力了，这个时候就需要设计高性能的集群来应对。

本次架构演进的方向正是引入负载均衡实现集群部署，实现方式如下：

使用Web集群方式部署之后，首要调整的就是用户状态信息与附件信息。用户状态不能再保存到Session中，缓存也不能用本地Web服务器的文件缓存。以及附件也不能保存在Web应用服务器上了。因为要保证集群里面的各个Web服务器，状态完全一致。因此，需要将用户状态、缓存等保存到专用的缓存服务器，比如Memcache。附件需要保存到云存储中，比如七牛云存储、阿里云存储、腾讯云存储等。

### 1、缓存服务器
集中设置一个缓存服务器，然后对应用进行一些代码层面的改造，将Session、缓存等保存到缓存服务器上。

- 自己搭建。找一台ecs实例分别安装memcache、redis数据库。所有的应用都连到这个缓存服务器上（业务量上来，下下一步考虑如何升级为分布式缓存服务）。
- 使用云数据库。云数据库Memcache版、云数据库Redis版。

着手搭建缓存服务器时，在纠结到底是用memcache还是memcached及其区别的时候，有朋友建议：为何不用redis呢？我的回答是看到thinkphp框架源码写着支持memcache，我本着以最少的修改，和时间成本来处理，毕竟系统目前在稳定运行中。但我还是查了下官方手册，这一查发现tp3可以使用redis作为缓存。数据缓存、查询缓存都可以。session的话有memcache驱动，redis就要自己写驱动。快速缓存（F方法）除了File本地还有Sae的驱动。

实现Redis缓存及Session:
只要在\ThinkPHP\Conf\convention.php文件里修改DATA_CACHE_TYPE为Redis，SESSION_OPTIONS以及SESSION_TYPE为Redis

但我们一般不直接修改TP内核的文件，直接在\Application\Common\Conf\config.php文件里面添加配置项：
```
'DATA_CACHE_TYPE' => 'Redis',
    'SESSION_OPTIONS' => array(
        'type' => 'Redis',
        'prefix' => 'sess_',
        'path' => 'tcp://127.0.0.1:6379?auth=',
        'expire' => 3600,
    ),
    'SESSION_TYPE' => 'Redis', // session hander类型 默认无需设置 除非扩展了session hander驱动
    'SESSION_PREFIX' => 'sess_', // session 前缀
    'VAR_SESSION_ID' => 'session_id', //sessionID的提交变量
    'SESSION_PERSISTENT' => 1, //是否长连接(对于php来说0和1都一样)
    'SESSION_CACHE_TIMEOUT' => 3, //连接超时时间(秒)
    'SESSION_REDIS_EXPIRE' => 3600, //session有效期(单位:秒) 0表示永久缓存
    'SESSION_REDIS_HOST' => '127.0.0.1', //redis服务器ip
    'SESSION_REDIS_PORT' => '6379', //端口
    'SESSION_REDIS_AUTH' => '', //认证密码
    'SESSION_REDIS_SELECT' => '1', //操作库
```

需要新增一个驱动文件。路径如下：ThinkPHP/Library/Think/Session/Driver/Redis.class.php
```
<?php
// +----------------------------------------------------------------------
// | ThinkPHP [ WE CAN DO IT JUST THINK ]
// +----------------------------------------------------------------------
// | Copyright (c) 2006~2017 http://thinkphp.cn All rights reserved.
// +----------------------------------------------------------------------
// | Licensed ( http://www.apache.org/licenses/LICENSE-2.0 )
// +----------------------------------------------------------------------
// | Author: liu21st <liu21st@gmail.com>
// +----------------------------------------------------------------------
namespace Think\Session\Driver;

use think\Exception;

class Redis implements \SessionHandlerInterface
{
    /** @var \Redis */
    protected $handler = null;
    protected $config = [
        'host' => '127.0.0.1',
        // redis端口
        'port' => 6379,
        // 密码
        'password' => '123456',
        // 操作库
        'select' => 1,
        // 有效期(秒)
        'expire' => 3600,
        // 超时时间(秒)
        'timeout' => 0,
        // 是否长连接
        'persistent' => true,
        // sessionkey前缀
        'session_name' => 'session_',
    ];

    public function __construct($config = [])
    {
        $this->config['host'] = C("SESSION_REDIS_HOST") ? C("SESSION_REDIS_HOST") : $this->config['host'];
        $this->config['port'] = C("SESSION_REDIS_POST") ? C("SESSION_REDIS_POST") : $this->config['port'];
        $this->config['password'] = C("SESSION_REDIS_AUTH") ? C("SESSION_REDIS_AUTH") : $this->config['password'];
        $this->config['select'] = C("SESSION_REDIS_SELECT") ? C("SESSION_REDIS_SELECT") : $this->config['select'];
        $this->config['expire'] = C("SESSION_REDIS_EXPIRE") ? C("SESSION_REDIS_EXPIRE") : $this->config['expire'];
        $this->config['session_name'] = C('SESSION_PREFIX') ? C('SESSION_PREFIX') : $this->config['session_name'];
        $this->config['timeout'] = C('SESSION_CACHE_TIMEOUT') ? C('SESSION_CACHE_TIMEOUT') : $this->config['timeout'];
    }

    /**
     * 打开Session
     * @access public
     * @param string $savePath
     * @param mixed $sessName
     * @return bool
     * @throws Exception
     */
    public function open($savePath, $sessName)
    {
        // 检测php环境
        if (!extension_loaded('redis')) {
            throw new Exception('not support:redis');
        }
        $this->handler = new \Redis;
        // 建立连接
        $func = $this->config['persistent'] ? 'pconnect' : 'connect';
        $this->handler->$func($this->config['host'], $this->config['port'], $this->config['timeout']);
        if ('' != $this->config['password']) {
            $this->handler->auth($this->config['password']);
        }
        if (0 != $this->config['select']) {
            $this->handler->select($this->config['select']);
        }
        return true;
    }

    /**
     * 关闭Session
     * @access public
     */
    public function close()
    {
        $this->gc(ini_get('session.gc_maxlifetime'));
        $this->handler->close();
        $this->handler = null;
        return true;
    }

    /**
     * 读取Session
     * @access public
     * @param string $sessID
     * @return string
     */
    public function read($sessID)
    {
        return (string)$this->handler->get($this->config['session_name'] . $sessID);
    }

    /**
     * 写入Session
     * @access public
     * @param string $sessID
     * @param String $sessData
     * @return bool
     */
    public function write($sessID, $sessData)
    {

        if ($this->config['expire'] > 0) {
            return $this->handler->setex($this->config['session_name'] . $sessID, $this->config['expire'], $sessData);
        } else {
            return $this->handler->set($this->config['session_name'] . $sessID, $sessData);
        }
    }

    /**
     * 删除Session
     * @access public
     * @param string $sessID
     * @return bool
     */
    public function destroy($sessID)
    {
        return $this->handler->delete($this->config['session_name'] . $sessID) > 0;
    }

    /**
     * Session 垃圾回收
     * @access public
     * @param string $sessMaxLifeTime
     * @return bool
     */
    public function gc($sessMaxLifeTime)
    {
        return true;
    }
}

```
至此解决了使用redis实现数据缓存及session。但是应用数据、应用日志、应用模板缓存还没有解决。
### 2、云存储
同样，原本应用上传下载等附件的操作方式也需要进行修改，改为保存/读取云存储的。

初步选用七牛云存储服务。

查阅官方手册可以得知，tp3目前已经支持的上传驱动包括Local、Ftp、Sae、Bcs、七牛和又拍云等。所以只需修改配置文件就能实现上传七牛云。
在\Application\Common\Conf\config.php文件里面添加配置项：
```
'UPLOAD_QINIU' => array(
    'maxSize' => 5 * 1024 * 1024,//文件大小
    'rootPath' => './',
    'saveName' => array('uniqid', ''),
    'driver' => 'Qiniu',
    'driverConfig' => array(
        'accessKey' => QINIU_ACCESS_KEY,
        'secretKey' => QINIU_SECRET_KEY,
        'domain' => QINIU_DOMAIN,
        'bucket' => QINIU_BUCKET,
        'up_host' => 'http://up-z2.qiniu.com' //上传地址
    )
)
```
这里注意官方还是华东的上传地址，需要手动修改QiniuStorage.class.php文件的配置，修改上传地址：
ThinkPHP/Library/Think/Upload/Driver/Qiniu/QiniuStorage.class.php

```
// 控制器里面增加上传文件方法
public function upload()
{
    $setting = C('UPLOAD_QINIU');
    $upload = new \Think\Upload($setting);
    $info = $upload->upload($_FILES);
    if(!$info) {
        // 上传错误提示错误信息
        echo 'error';
        $this->error($upload->getError());
    }else{
        // 上传成功 获取上传文件信息
        foreach($info as $file){
            echo $file['savepath'].$file['savename'];
        }
    }
}
```

### 3、集群部署和负载均衡
经过以上两项的修改，就可以使用负载均衡来实现大规模集群方式架设系统。负载均衡实现的方案有很多，自己搭建也比较考验运维动手能力。所以这里建议选用阿里的[负载均衡SLB](https://www.aliyun.com/product/slb)产品。

阿里云负载均衡SLB包含两款产品类型，分别为应用型负载均衡ALB和传统型负载均衡CLB：
- 应用型负载均衡ALB：聚焦HTTP、HTTPS和QUIC应用层协议，面向应用交付。具备超高性能、超大弹性、丰富7层转发特性、集成WAF/云原生等能力。
- 传统型负载均衡CLB：支持TCP/UDP/HTTP/HTTPS等协议，主要面向网络交付。提供性能保障实例、高安全、高可靠、即开即用、全场景IPv6等能力。

**我们系统是应用型负载，故选择应用型负载均衡ALB。**
#### 应用型负载均衡ALB-产品价格详情
ALB目前仅支持按量付费。ALB费用由三部分组成：实例费、性能容量单位LCU（Loadbalancer Capacity Unit）费和公网网络费。[具体参考链接](https://www.aliyun.com/price/product?spm=5176.7921785.J_5253785160.6.70a62229XnQsUE#/slb/detail/slb)


