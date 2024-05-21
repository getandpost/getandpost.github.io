# GatewayWorker升级后报警告推送不了问题

### 背景
使用GatewayWorker做的一个PC端消息推送功能，在执行composer update升级后，推送功能使用不了了。
```
$ composer update
Loading composer repositories with package information
Updating dependencies
Lock file operations: 2 installs, 0 updates, 0 removals
  - Locking workerman/gateway-worker (v3.1.17)
  - Locking workerman/workerman (v4.1.15)
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 0 installs, 2 updates, 0 removals
  - Downloading workerman/gateway-worker (v3.1.17)
  - Upgrading workerman/workerman (v4.0.18 => v4.1.15): Extracting archive
  - Upgrading workerman/gateway-worker (v3.0.19 => v3.1.17): Extracting archive
1 package suggestions were added by new dependencies, use `composer suggest` to see details.
Generating autoload files
2 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
```


### 故障分析
查看队列并无异样，也在消费。然后使用命令启动，不要-d：
```
[xxx-deploy@localhost GatewayWorker]$ sudo php start.php start
Workerman[start.php] start in DEBUG mode
------------------------------------------------ WORKERMAN ------------------------------------------------
Workerman version:4.1.15          PHP version:7.4.33           Event-Loop:\Workerman\Events\Select
------------------------------------------------- WORKERS -------------------------------------------------
proto   user            worker                   listen                      processes    status
tcp     root            YourAppBusinessWorker    none                        4             [OK]
tcp     root            YourAppGateway           websocket://0.0.0.0:8282    2             [OK]
tcp     root            Register                 text://127.0.0.1:1238       1             [OK]
-----------------------------------------------------------------------------------------------------------
Press Ctrl+C to stop. Start success.
Waring: Events::onMessage is not callable
Waring: Events::onMessage is not callable
Waring: Events::onMessage is not callable
Waring: Events::onMessage is not callable
```
可以发现控制台报错：Waring: Events::onMessage is not callable了。

原因可能是Applications目录没有自动加载。需要把Applications目录放在composer.json 文件中。


### 解决方案
修改composer.json文件，把autoload添加进去：
```
{
    "name"  : "workerman/gateway-worker-demo",
    "keywords": ["distributed","communication"],
    "homepage": "http://www.workerman.net",
    "license" : "MIT",
    "require": {
        "workerman/gateway-worker" : ">=3.0.0"
    },
    "autoload": {
        "psr-4": {
            "" : "./",
            "" : "./Applications/YourApp/"
        }
    }
}

```
然后执行
```
composer dump-autoload
```

重新启动，问题解决。
```
[xxx-deploy@localhost GatewayWorker]$ sudo php start.php start
Workerman[start.php] start in DEBUG mode
------------------------------------------------ WORKERMAN ------------------------------------------------
Workerman version:4.1.15          PHP version:7.4.33           Event-Loop:\Workerman\Events\
------------------------------------------------- WORKERS -------------------------------------------------
proto   user            worker                   listen                      processes    status
tcp     root            YourAppBusinessWorker    none                        4             [OK]
tcp     root            YourAppGateway           websocket://0.0.0.0:8282    2             [OK]
tcp     root            Register                 text://127.0.0.1:1238       1             [OK]
-----------------------------------------------------------------------------------------------------------
Press Ctrl+C to stop. Start success.
```