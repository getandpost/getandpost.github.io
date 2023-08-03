# Ubuntu下使用SSH连接CentOS系统问题

### 背景
工作中用的WSL2子系统，安装的是Ubuntu 22.04.2 LTS版本。局域网内部服务器安装的是CentOS Linux release 7.9.2009 (Core)版本。

### 遇到的问题
- 1、Ubuntu下使用SSH连接CentOS系统，发现连接很慢。普遍要35s以上。心想这是局域网不至于啊。
- 2、Ubuntu下使用SSH连接CentOS系统，发现vim编辑文件时，中文为乱码，很是不方便；

### 解决方案
Ubuntu下使用SSH连接CentOS系统慢的问题，网上说是Ubuntu的一个Bug。解决此Bug很简单，只需要修改/etc/ssh/ssh_config即可。如下：
```
sudo vim /etc/ssh/ssh_config
```
把ssh_config配置文件中GSSAPIAuthentication由原来的yes修改为no即可。

ubuntu ssh centos vim中文乱码问题是两边编码不一致导致的。

先看本地语言设置
```
echo $LANG
```
结果显示为C.UTF-8。再看终端的语言设置，结果显示为en_US.UTF-8 linux C.UTF-8和en-US.UTF-8语言环境有什么区别？网上有好多种修改方式，综合比对并且思考了一下，我使用的解决方法如下：
```
sudo vim /etc/default/locale
```
修改locale文件，把LANG=C.UTF-8改为LANG=en_US.UTF-8