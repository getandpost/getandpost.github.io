# 2023-02-16-Linux系统多版本php安装swoole等扩展

### 一、首先使用源码编译安装：
```
/usr/local/php74/bin/phpize
./configure --with-php-config=/usr/local/php74/bin/php-config
```
编译安装完成后验证，出现报错信息：
```
php -m | grep swoole
PHP Warning:  PHP Startup: Unable to load dynamic library 'swoole.so' (tried: /usr/local/php74/lib/php/extensions/no-debug-non-zts-20190902/swoole.so (/usr/local/php74/lib/php/extensions/no-debug-non-zts-20190902/swoole.so: undefined symbol: _ZTINSt6thread6_StateE), /usr/local/php74/lib/php/extensions/no-debug-non-zts-20190902/swoole.so.so (/usr/local/php74/lib/php/extensions/no-debug-non-zts-20190902/swoole.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
```
网上查询有说是swoole.so放的位置不对。最后发现是gcc的版本不对。以前编译php7用的是老版本的gcc，编译swoole用的新版本的gcc，两个版本的支持库不兼容，所以只能是两种方法，一是降gcc的版本，二是用新版本的gcc重新编译php7。


### 二、使用pecl安装
网上查询 linux系统，多版本php环境使用PECL指定php版本安装，使用pecl安装swoole、redis等
尝试pecl
```
pecl install swoole-4.8.12

Warning: readlink() has been disabled for security reasons in Guess.php on line 231
PHP Warning:  readlink() has been disabled for security reasons in /usr/local/php/lib/php/OS/Guess.php on line 231

Warning: popen() has been disabled for security reasons in Guess.php on line 306
PHP Warning:  popen() has been disabled for security reasons in /usr/local/php/lib/php/OS/Guess.php on line 306

Warning: fgets() expects parameter 1 to be resource, null given in Guess.php on line 307
PHP Warning:  fgets() expects parameter 1 to be resource, null given in /usr/local/php/lib/php/OS/Guess.php on line 307

Warning: pclose() expects parameter 1 to be resource, null given in Guess.php on line 316
PHP Warning:  pclose() expects parameter 1 to be resource, null given in /usr/local/php/lib/php/OS/Guess.php on line 316
pecl/swoole requires PHP (version >= 7.2.0), installed version is 5.6.40
No valid packages found
install failed
```

查看pecl的版本信息，
```
pecl version
PEAR Version: 1.10.13
PHP Version: 5.6.40
Zend Engine Version: 2.6.0
Running on: Linux izwz9ioqf1qlimew3x9bwpz 3.10.0-693.2.2.el7.x86_64 #1 SMP Tue Sep 12 22:26:13 UTC 2017 x86_64
```

如何配置pecl的php版本
```
pecl config-show

pecl config-set php_bin /usr/local/php74/bin/php
pecl config-set php_dir /usr/local/php74/lib/php
pecl config-set ext_dir /usr/local/php74/lib/php/extensions/no-debug-non-zts-20190902
pecl config-set php_ini /usr/local/php74/etc/php.ini
pecl config-set bin_dir /usr/local/php74/bin
```

再次执行pecl install swoole-4.8.12安装，还是报同样的错误。
```
pecl -d php_suffix=7.4 install swoole-4.8.12
```
结果还是一样的错误。于是就想到升级pecl吧，升到跟当前运行的php7.4一样。

### 三、升级pecl
PECL是PHP扩展包仓库，方便扩展包开发和下载，这些扩展一般都是C语言开发的，默认仓库是pecl.php.net
常用命令：
```
pecl version #查看版本
pecl help #查看帮助信息
pecl config-show #查看配置
```

PEAR (PHP Extension and Application Repository)
PHP扩展和应用仓库，既有PHP扩展（基于C，和pecl共用仓库），又有PHP类库（基于PHP）
常用命令：
```
pear version #查看版本
pear help #查看帮助信息
pear config-show #查看配置
```

php -v 看到的版本和 pear version显示的版本不一致的问题，重新安装pear即可【[安装官网：](https://pear.php.net/manual/en/installation.getting.php)】（pecl随之更新）。

```
wget http://pear.php.net/go-pear.phar
php go-pear.phar
```

执行pecl version查看发现还有个别参数是php5.6的，需要手动更换配置：
```
/usr/local/php74/bin/pecl config-set php_bin /usr/local/php74/bin/php
/usr/local/php74/bin/pecl config-set php_dir /usr/local/php74/lib/php
/usr/local/php74/bin/pecl config-set ext_dir /usr/local/php74/lib/php/extensions/no-debug-non-zts-20190902
/usr/local/php74/bin/pecl config-set php_ini /usr/local/php74/etc/php.ini
/usr/local/php74/bin/pecl config-set bin_dir /usr/local/php74/bin
```

再次执行命令安装：
```
/usr/local/php74/bin/pecl install swoole-4.8.12
```
终于看到下载并跑起安装流程来了，有点小激动。最后还是泼了冷水，报错信息如下：
```
checking for cc option to accept ISO C99... none needed
/tmp/pear/install/swoole/configure: line 6086: syntax error near unexpected token `-Wbool-conversion,'
/tmp/pear/install/swoole/configure: line 6086: `        AX_CHECK_COMPILE_FLAG(-Wbool-conversion,                _MAINTAINER_CFLAGS="$_MAINTAINER_CFLAGS -Wbool-conversion")'
ERROR: `/tmp/pear/install/swoole/configure --with-php-config=/usr/local/php74/bin/php-config --enable-sockets=y --enable-openssl=y --enable-http2=y --enable-mysqlnd=y --enable-swoole-json=y --enable-swoole-curl=y --enable-cares=y' failed
```

当前版本的swoole要求的编译器与现在系统上的编译器版本不兼容。目前系统的gcc不支持 -Wbool-conversion 这个选项。
```
gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/local/libexec/gcc/x86_64-pc-linux-gnu/11.3.0/lto-wrapper
Target: x86_64-pc-linux-gnu
Configured with: ../gcc-11.3.0/configure --enable-bootstrap --enable-checking=release --enable-languages=c,c++,go --disable-multilib
Thread model: posix
Supported LTO compression algorithms: zlib
gcc version 11.3.0 (GCC)
```

果然问题还是出现在gcc版本上面。在另外一台生产服务器，gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 是可以使用源码编译安装成功的。

(安装不成功，未完待续....)

### PS:一个小插曲，关于我本地机的
本机是ubuntu最新版本，最新源。目前有php7.4和php8.1两个版本。其实我只想要7.4版，但系统自带了8.1版本。

```
pecl -d php_suffix=7.4 install http://pecl.php.net/get/swoole-4.8.12.tgz
```
出现如下报错信息:
```
WARNING: channel "pecl.php.net" has updated its protocols, use "pecl channel-update pecl.php.net" to update
downloading swoole-4.8.12.tgz ...
Starting to download swoole-4.8.12.tgz (853,688 bytes)
.........................done: 853,688 bytes
248 source files, building
running: phpize
sh: 1: phpize7.4: not found
ERROR: `phpize' failed
```

执行命令查看phpize情况：
```
ll /usr/bin/php*
lrwxrwxrwx 1 root root      21 Dec  5 16:38 /usr/bin/php -> /etc/alternatives/php*
lrwxrwxrwx 1 root root      28 Feb  8 11:49 /usr/bin/php-config -> /etc/alternatives/php-config*
-rwxr-xr-x 1 root root    4356 Feb 15 02:35 /usr/bin/php-config8.1*
-rwxr-xr-x 1 root root 4813912 Feb 15 02:31 /usr/bin/php7.4*
-rwxr-xr-x 1 root root 5568008 Feb 15 02:35 /usr/bin/php8.1*
-rwxr-xr-x 1 root root    8676 Nov  2  2021 /usr/bin/phpabtpl*
lrwxrwxrwx 1 root root      24 Feb  8 11:49 /usr/bin/phpize -> /etc/alternatives/phpize*
-rwxr-xr-x 1 root root    4948 Feb 15 02:35 /usr/bin/phpize8.1*
```

发现只有php8.1的，尝试执行命令安装php7.4的：
```
apt search php-dev 
sudo apt install php7.4-dev
```

再次查看：
```
ll /usr/bin/php*
lrwxrwxrwx 1 root root      21 Dec  5 16:38 /usr/bin/php -> /etc/alternatives/php*
lrwxrwxrwx 1 root root      28 Feb  8 11:49 /usr/bin/php-config -> /etc/alternatives/php-config*
-rwxr-xr-x 1 root root    4363 Feb 15 02:31 /usr/bin/php-config7.4*
-rwxr-xr-x 1 root root    4356 Feb 15 02:35 /usr/bin/php-config8.1*
-rwxr-xr-x 1 root root 4813912 Feb 15 02:31 /usr/bin/php7.4*
-rwxr-xr-x 1 root root 5568008 Feb 15 02:35 /usr/bin/php8.1*
-rwxr-xr-x 1 root root    8676 Nov  2  2021 /usr/bin/phpabtpl*
lrwxrwxrwx 1 root root      24 Feb  8 11:49 /usr/bin/phpize -> /etc/alternatives/phpize*
-rwxr-xr-x 1 root root    4932 Feb 15 02:31 /usr/bin/phpize7.4*
-rwxr-xr-x 1 root root    4948 Feb 15 02:35 /usr/bin/phpize8.1*
```

再次执行：
```
pecl -d php_suffix=7.4 install swoole-4.8.12
```
显示已经安装过了。又是上网一通查询，发现解决方法：
```
pecl uninstall -r swoole-4.8.12
pecl -d php_suffix=7.4 install swoole-4.8.12
```
大概意思是当系统中安装了多个php版本时，使用pecl为特定的php版本时需要uninstall -r 再安装一次。逐个版本依此类推。至此本地机安装成功，处理一下php.ini的swoole.so即可使用。