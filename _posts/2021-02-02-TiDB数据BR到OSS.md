## TiDB集群BR到阿里云的OSS
其实最简单的方法是三台tikv机器购买数据盘挂载作为备份盘。为了省钱，只能折腾。这里使用的是阿里云OSS来存储。

利用官方提供的ossfs在Linux系统中，将对象存储OSS的存储空间（Bucket）挂载到本地文件系统中，能够像操作本地文件一样操作OSS的对象（Object），实现数据的共享。

参考阿里官方文档：对象存储 OSS > 常用工具 > ossfs > [快速安装](https://help.aliyun.com/document_detail/153892.html?spm=a2c4g.11186623.6.919.75202b4fdhzMdH)

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
echo xxx-cluster:LTAIbZcdVCmQ****:MOk8x0y9hxQ31coh7A5e2MZEUz**** > ~/passwd-ossfs
chmod 640 ~/passwd-ossfs
ossfs xxx-cluster/tidbbak /ossfs -ourl=http://oss-cn-shenzhen.aliyuncs.com
```

5.如果您不希望继续挂载此Bucket，您可以将其卸载。
```
fusermount -u /ossfs
```

### 使用BR备份集群
```
su tidb
mkdir -p /ossfs/20220202
br backup table --pd "10.0.0.80:2379" --db xxx --table t -s "local:///ossfs/tidbbak/20220202" --log-file backup-nfs.log
```