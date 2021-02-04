## TiDB集群BR到阿里云的OSS
其实最简单的方法是三台tikv机器购买数据盘挂载作为备份盘。为了省钱，只能折腾。这里使用的是阿里云OSS来存储。

利用官方提供的ossfs在Linux系统中，将对象存储OSS的存储空间（Bucket）挂载到本地文件系统中，能够像操作本地文件一样操作OSS的对象（Object），实现数据的共享。

参考阿里官方文档：对象存储 OSS > 常用工具 > ossfs > [快速安装](https://help.aliyun.com/document_detail/153892.html?spm=a2c4g.11186623.6.919.75202b4fdhzMdH)

文档高级配置部分要仔细阅读。

### 运行环境
Ubuntu 20.04

### 给每台kv主机安装ossfs并挂载
1.下载安装包。
```
wget http://gosspublic.alicdn.com/ossfs/ossfs_1.80.6_ubuntu18.04_amd64.deb
```
这里没找到20.04对应的版本，只能用Ubuntu 18.04 (x64)对应的。

2.安装ossfs。
Ubuntu系统的安装命令：
```
sudo apt-get update
sudo apt-get install gdebi-core
sudo gdebi ossfs_1.80.6_ubuntu18.04_amd64.deb
```

3.配置账号访问信息。
将Bucket名称以及具有此Bucket访问权限的AccessKeyId/AccessKeySecret信息存放在/etc/passwd-ossfs文件中。注意这个文件的权限必须正确设置，建议设为640。
```
echo my-bucket:my-access-key-id:my-access-key-secret > /etc/passwd-ossfs
chmod 640 /etc/passwd-ossfs
```


4.将Bucket挂载到指定目录。
```
ossfs my-bucket my-mount-point -ourl=my-oss-endpoint
```

挂载示例：将深圳地域名称为xxx-cluster的Bucket挂载到/ossfs目录下。
```
mkdir -p /ossfs
chown -R tidb:tidb /ossfs
su tidb
cd ~
echo xxx-cluster:LTAIbZcdVCmQ****:MOk8x0y9hxQ31coh7A5e2MZEUz**** > ~/.passwd-ossfs
chmod 600 ~/.passwd-ossfs
ossfs xxx-cluster /ossfs -ourl=http://oss-cn-shenzhen.aliyuncs.com
```

5.如果您不希望继续挂载此Bucket，您可以将其卸载。
```
fusermount -u /ossfs
```
遇到fusermount: failed to unmount : Device or resource busy错误可以使用：
```
fusermount -zu /ossfs
```

### 使用BR备份集群
**注意：** br版本要跟tidb集群的版本一致。执行 BR 的用户和启动 TiKV 的用户要相同。
```
su tidb
mkdir -p /ossfs/20220202
#备份数据库
br backup table --pd "10.0.0.80:2379" --db xxx --table t -s "local:///ossfs/tidbbak/20220202" --log-file backup-nfs.log

#备份集群
./br backup full --pd 10.0.0.80:2379 -s "local:///tidbbak/full_20210203" --log-file backupdb_full.log
```
如果是OSS挂载方式，在OSS文件管理删除了backup.lock后，还会报Error: backup lock exists, may be some backup files in the path already: [BR:Common:ErrInvalidArgument]invalid argument错误。先fusermount再重新ossfs挂载即可。

踩坑一：执行BR过程中把少量文件写入oss桶，就报错Database backup <-…> 0.12%
Error: msg:“Io(Os { code: 1, kind: PermissionDenied, message: “Operation not permitted” })” : [BR:KV:ErrKVUnknown]unknown tikv error
后来发现：可以echo “aaa” > /dbbak/test.log，但是不能rm /dbbak/test.log rm: cannot remove ‘/dbbak/test.log’: Operation not permitted

在想我的/dbbak目录权限是drwxrwxrwx，会不会不够？按理应该是777了的。
最后跑到阿里云打开oss管理控制台，果然我的Bucket 授权策略为 读写，我现在改成完全控制就可以rm了。解决了rm问题，就可以顺利地执行BR到OSS了。

### 挂载脚本
root用户下创建ossfs.sh脚本并执行
```
#!/bin/sh

set -e

#配置好oss参数
AccessKey=OSS的access-key-id
SecretKey=OSS的access-key-secret
bucketName='bucket名称'
Endpoint=http://oss-cn-shenzhen-internal.aliyuncs.com   
#Endpoint地址请参考：https://help.aliyun.com/document_detail/31837.html?spm=a2c4g.11186623.6.626.581e7a6cMIuIYO

#开始挂载
mkdir -p /dbbak
echo "${bucketName}:${AccessKey}:${SecretKey}" > /etc/passwd-ossfs
chmod 640 /etc/passwd-ossfs
ossfs ${bucketName} /dbbak -ourl=${Endpoint} -oallow_other

#开机启动
echo '#! /bin/bash' > /etc/init.d/ossfs
echo '# chkconfig: 2345 90 10' >> /etc/init.d/ossfs
echo "ossfs ${bucketName} /dbbak -ourl=${Endpoint} -oallow_other" >> /etc/init.d/ossfs
chmod a+x /etc/init.d/ossfs
chkconfig ossfs on
```

### 执行备份脚本
```
su - tidb
mkdir -p /dbbak/full_20210204
br backup table --pd 10.0.0.80:2379 --db "xxx" --table "xxx_admin" --storage local:///dbbak/full_20210204 --ratelimit 120 --log-file backuptable.log
```
