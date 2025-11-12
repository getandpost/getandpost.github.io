# php-resque使用问题

## 问题描述
php项目里面用的是一个轻量级的队列[resque](https://github.com/resque/php-resque).最近发现应用总是往queue:failed里面写数据，查看/home/data/supervisor_log/aems-resque.out.log日志，都显示任务已经执行结束 has finished了。进入redis集群使用命令lrange queue:failed -1 -1 查看日志，里面信息都是无一都显示：\"exception\":\"Resque_Job_DirtyExitException\",\"error\":\"Job exited abnormally\",\"backtrace\" 这个错误。这是什么原因？

## 原因分析
Resque_Job_DirtyExitException 和 "Job exited abnormally" 错误通常是由于 Worker 进程在任务执行过程中异常退出导致的，即使任务实际上已经成功执行完成。
有可能是内存限制导致进程被杀死。

### 排查步骤1：检查内存限制
```bash
# 检查系统日志看是否有 OOM 记录
dmesg | grep -i "killed process"
# 或者
grep -i "out of memory" /var/log/syslog
```
虽然没看到有相关的OOM信息，顺便检查PHP-FPM的配置文件php.ini设置memory_limit的大小确实只有128M，感觉太小了。
尝试增加内存的解决方案：
```bash
[program:xxx-resque]
command = /usr/local/php74/bin/php -d memory_limit=1024M resque.php start --queue=jpush,xxx_queue --interval=10
```
因为不敢直接修改php.ini，所以直接在supervisor的配置文件中增加memory_limit的设置。重启队列后发现问题依然存在。

### 排查步骤2：检查 Supervisor 配置
现在有点怀疑问题是出在使用了Supervisor管理队列进程，而不是直接用命令行启动。
```bash
[program:aems-resque]
command=/usr/local/php74/bin/php -d memory_limit=1024M resque.php start --queue=jpush,xxx_queue --interval=10
environment=ASPNETCORE__ENVIRONMENT=Production
user=www
process_name=%(program_name)s_%(process_num)02d
numprocs=5
directory=/path/to/your/project

; 关键修改：使用 QUIT 信号优雅退出
stopsignal=QUIT
; 增加:停止等待时间，让任务有足够时间完成
stopwaitsecs=300

; 防止频繁重启
autostart=true
autorestart=true
startsecs=1
startretries=3

; 日志配置
stderr_logfile=/home/data/supervisor_log/xxx-resque.err.log
stdout_logfile=/home/data/supervisor_log/xxx-resque.out.log
stdout_logfile_maxbytes=50MB
stderr_logfile_maxbytes=50MB

; 增加:防止僵尸进程
stopasgroup=true
killasgroup=true
```
执行命令重启队列：
```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart aems-resque:*
```
观察结果，发现还是有异常记录。接着怀疑一个 Worker 监听太多队列，某些队列任务得不到及时处理。

### 排查步骤3：检查代码逻辑
不得不怀疑是业务出错，并且Job 类中不能使用 exit() 或 die()，必须使用 return 或 throw Exception。查了一遍业务，业务是没有问题的。确定是 Worker 进程管理的问题，不是业务代码的问题。
问题应该出在resque源码。于是从头阅读源码排查，发现队列是依赖PCNTL扩展的，而PCNTL扩展默认是不开启的。相起年初的时候我们项目迁移过云服务器。

修改vendor/resque/php-resque/lib/Resque/Worker.php文件，在fork子进程后增加日志记录功能：
```php
if($this->child > 0) {
    // Parent process, sit and wait
    $status = 'Forked ' . $this->child . ' at ' . strftime('%F %T');
    $this->updateProcLine($status);
    $this->logger->log(Psr\Log\LogLevel::INFO, $status);

    $logFile = '/home/data/supervisor_log/resque_debug.log';
    // 记录 fork 信息
    file_put_contents($logFile, sprintf(
        "[%s] Forked child %d for job: %s (ID: %s)\n",
        date('Y-m-d H:i:s'),
        $this->child,
        $job->payload['class'],
        $job->payload['id']
    ), FILE_APPEND);

    // Wait until the child process finishes before continuing
    while (pcntl_wait($status, WNOHANG) === 0) {
        if(function_exists('pcntl_signal_dispatch')) {
            pcntl_signal_dispatch();
        }

        // Pause for a half a second to conserve system resources
        usleep(500000);
    }

    // 记录详细的退出状态
    file_put_contents($logFile, sprintf(
        "[%s] Child %d exit status: %d\n" .
        "  - WIFEXITED: %s\n" .
        "  - WIFSIGNALED: %s\n" .
        "  - WTERMSIG: %s\n" .
        "  - WEXITSTATUS: %s\n" .
        "  - Job: %s\n\n",
        date('Y-m-d H:i:s'),
        $this->child,
        $status,
        pcntl_wifexited($status) ? 'true' : 'false',
        pcntl_wifsignaled($status) ? 'true' : 'false',
        pcntl_wifsignaled($status) ? pcntl_wtermsig($status) : 'N/A',
        pcntl_wifexited($status) ? pcntl_wexitstatus($status) : 'N/A',
        $job->payload['class']
    ), FILE_APPEND);

    if (pcntl_wifexited($status) !== true) {
        // 记录异常退出
        file_put_contents($logFile, sprintf(
            "[%s] ERROR: Job exited abnormally! Child was %s\n",
            date('Y-m-d H:i:s'),
            pcntl_wifsignaled($status) ? 'killed by signal ' . pcntl_wtermsig($status) : 'terminated abnormally'
        ), FILE_APPEND);
        $job->fail(new Resque_Job_DirtyExitException('Job exited abnormally'));
    } elseif (($exitStatus = pcntl_wexitstatus($status)) !== 0) {
        $job->fail(new Resque_Job_DirtyExitException(
            'Job exited with exit code ' . $exitStatus
        ));
    }
    else
    {
        if (in_array($job->getStatus(), array(Resque_Job_Status::STATUS_WAITING, Resque_Job_Status::STATUS_RUNNING)))
        {
            $job->updateStatus(Resque_Job_Status::STATUS_COMPLETE);
            $this->logger->log(Psr\Log\LogLevel::INFO, 'done ' . $job);
        }
    }
}
```

测试结果：
```
[2025-11-11 17:50:39] ERROR: Job exited abnormally! Child was terminated abnormally
[2025-11-11 17:50:39] Forked child 275627 for job: \Common\Job\PushJob (ID: 7c71c33bf7d124ccdccc58b3c03905a3)
[2025-11-11 17:50:40] Child 275627 exit status: 0
  - WIFEXITED: false
  - WIFSIGNALED: false
  - WTERMSIG: N/A
  - WEXITSTATUS: N/A
  - Job: \Common\Job\PushJob
```

### 排查步骤4：检查PCNTL扩展
```bash
# 查看 PHP 配置文件是否启用了 pcntl 扩展
cat /usr/local/php74/etc/php.ini | grep -i "extension=pcntl"

# 如果未启用，需要取消注释或添加以下行到 php.ini 文件：
; extension=pcntl.so  // 注意路径可能根据安装位置不同而变化

# 重启 PHP-FPM 服务以使更改生效（取决于你的服务器设置）
sudo systemctl restart php7.4-fpm
```

### 排查步骤5：测试PCNTL功能
```bash
# 查看 PHP 版本
/usr/local/php74/bin/php -v

# 查看 PCNTL 扩展版本
/usr/local/php74/bin/php -i | grep -A 5 pcntl

# 测试 pcntl_wait 行为
/usr/local/php74/bin/php -r '
$pid = pcntl_fork();
if ($pid == 0) {
    // 子进程
    exit(0);
} else {
    // 父进程
    $status = 0;
    pcntl_waitpid($pid, $status, 0);
    echo "Status: $status\n";
    echo "WIFEXITED: " . (pcntl_wifexited($status) ? "true" : "false") . "\n";
    echo "WEXITSTATUS: " . pcntl_wexitstatus($status) . "\n";
}
'
```

测试结果：

```bash
[xxx-deploy@xxx-app-01 supervisor_log]$ /usr/local/php74/bin/php -i | grep -A 5 pcntl
disable_functions => pcntl_waitpid,pcntl_wifexited,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wifcontinued,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal_get_handler,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_exec,pcntl_getpriority,pcntl_setpriority,pcntl_async_signals,pcntl_unshare, => pcntl_waitpid,pcntl_wifexited,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wifcontinued,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal_get_handler,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_exec,pcntl_getpriority,pcntl_setpriority,pcntl_async_signals,pcntl_unshare,
display_errors => Off => Off
display_startup_errors => Off => Off
doc_root => no value => no value
docref_ext => no value => no value
docref_root => no value => no value
--
pcntl

pcntl support => enabled

pcre

PCRE (Perl Compatible Regular Expressions) Support => enabled
PCRE Library Version => 10.35 2020-05-09
[xxx-deploy@xxx-app-01 supervisor_log]$ /usr/local/php74/bin/php -r '
$pid = pcntl_fork();
if ($pid == 0) {
    // 子进程
    exit(0);
} else {
    // 父进程
    $status = 0;
    pcntl_waitpid($pid, $status, 0);
    echo "Status: $status\n";
    echo "WIFEXITED: " . (pcntl_wifexited($status) ? "true" : "false") . "\n";
    echo "WEXITSTATUS: " . pcntl_wexitstatus($status) . "\n";
}
'
PHP Warning:  pcntl_waitpid() has been disabled for security reasons in Command line code on line 9
Status: 0
PHP Warning:  pcntl_wifexited() has been disabled for security reasons in Command line code on line 11
WIFEXITED: false
PHP Warning:  pcntl_wexitstatus() has been disabled for security reasons in Command line code on line 12
WEXITSTATUS:
```

这下找到根本原因了！！！关键的 PCNTL 函数被禁用了！
这就是为什么：
- pcntl_wifexited($status) 总是返回 false
- pcntl_wexitstatus($status) 返回空
- 所有任务都被标记为 DirtyExitException


## 解决方案：
1.为 CLI 使用单独的 php.ini（因为不想影响 PHP-FPM）

```bash
# 复制一份配置
cp /usr/local/php74/etc/php.ini /usr/local/php74/etc/php-cli.ini

# 编辑 CLI 专用配置
vim /usr/local/php74/etc/php-cli.ini

# 修改 disable_functions，启用 pcntl 函数
```
Resque 必须启用的函数：
- pcntl_waitpid - 等待子进程
- pcntl_wifexited - 检查是否正常退出
- pcntl_wexitstatus - 获取退出码
- pcntl_wifsignaled - 检查是否被信号杀死
- pcntl_wtermsig - 获取信号编号

2.在 supervisor 配置中通过 -d 参数启用

```bash
[program:resque]
command=/usr/local/php74/bin/php -d /usr/local/php74/etc/php-cli.ini resque.php start --queue=jpush --interval=10
```

## 总结
- 遇到此类应该多看看开源包的源码，多吃透一下就知道原因了。