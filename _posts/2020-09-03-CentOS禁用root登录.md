## CentOS禁用root登录

### CentOS 7.4 禁用root登录，提高系统安全性

### 查看centos中的用户和用户组
1. 用户列表文件：/etc/passwd/
2. 用户组列表文件：/etc/group
3. 查看系统中有哪些用户：cut -d : -f 1 /etc/passwd
4. 查看可以登录系统的用户：cat /etc/passwd | grep -v /sbin/nologin | cut -d : -f 1

### 创建一个新账号
```
# 创建用户
useradd new-username
# 设置相应密码
passwd new-userpasswd
```

### 给新账户sudo权限
```
# 1.增加 sudoers 文件的写的权限，默认为只读
chmod -v u+w /etc/sudoers

# 2.修改 sudoers 文件添加一行
# new-username        ALL=(ALL)       ALL
vim /etc/sudoers

# 3.删除 sudoers 的写的权限
chmod -v u-w /etc/sudoers

```

### 禁止root账户ssh登录
```
# 1.找到PermitRootLogin yes将yes改为no
vim /etc/ssh/sshd_config

# 2.重启sshd服务
# service sshd restart
systemctl restart sshd.service
```


### usermod命令的 -s使用方法
禁止用户ssh远程登录

需要用到：usermod -s /sbin/nologin 用户名

恢复：usermod -s /bin/bash 用户名

检验：查看etc下的passwd文件 /bin/bash 可以ssh登录；如果为：/sbin/nologin 则禁止了ssh登录