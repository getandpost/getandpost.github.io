### Linux rsync实现两服务器文件同步

1、两台服务器都安装rsync
```
yum -y install rsync
```

2、设置密码认证文件和配置文件

服务端配置文件
```
echo "rsync:123456" > /home/rsync/rsyncd.passwd
```
客户端配置文件
```
echo "123456" /home/rsync/passwd.txt
```
设置权限为只读
```
chmod 600 rsyncd.passwd
```

rsync配置文件
```
[root@www rsync]# vim /etc/rsyncd.conf
uid = root
gid = root
port = 873
use chroot = no
read only = no
list = no
max connections = 1
timeout = 600
log file = /home/rsync/log/rsyncd.log
pidfile = /home/rsync/pid/rsyncd.pid
lock file = /home/rsync/run/rsync.lock
[rsync]
path = /alidata/www/xxx/
comment = rsync
ignore errors
auth users =  rsync
secrets file = /home/rsync/rsync.pass
hosts allow = 192.168.10.2
hosts deny = *
```

3、以守护进程方式启动rsync服务器(客户端不需要启动，就是没有修改配置文件的客户端)   
a.使用yum命令安装时
```
rsync --daemon
```
b.使用编译安装时
```
rsync --daemon --config=/etc/rsyncd.conf
```
加入开机自启动
```
echo "/usr/local/rsync/bin/rsync --daemon --config=/etc/rsyncd.conf" >> /etc/rc.local
```
客户端同步更新文件
```
sudo /usr/bin/rsync -vzrtopg --progress --delete --password-file=/home/rsync/passwd.txt rsync@192.168.10.2::rsync /alidata/www/xxx
```

如果想要排除某些目录或文件
```
/usr/bin/rsync -vzrtopg --progress --delete --exclude=/Application/Runtime --password-file=/home/rsync/passwd.pass rsync@172.18.33.83::rsync /alidata/www/xxx
```