# 2021-10-15-使用TiCDC实时同步TiDB数据到备用逃生环境

## 背景
技术顾问给我们选型TiDB。项目在内网环境搭建了TiDB群集，经过数个月的内网测试。我们还是采取了大胆的做法，直接停机做好数据迁移启用新库，抛弃旧库。这是一种开弓没有回头路的做法。
众周知实施过程并不像一般人所想的那样，新库对上层系统的兼容性需要长时间的测试以及实际生产过程来证明。过程中一旦发生问题，想要恢复将会遇到大问题。其实保险点做法是平滑迁移。

何谓平滑迁移？也就是说并不在迁移阶段初始阶段就全部启用 TiDB。而是首先做好数据迁移，数据同步操作。接着将上层系统的读流量接入 TiDB，经过一段时间发现 TiDB 可以胜任读库任务，然后就在此基础上接入写流量。期间使用运维工具将 TiDB 的数据反向同步到 MySQL 以保证万一 TiDB 出现问题无法支持上层业务系统时，能够快速切换到该 MySQL 环境。我们称这个 MySQL 为逃生环境。

由于各种条件限制，一直没有实现。这让我们在刚上线阶段吃了大苦头，一度想放弃TiDB，重新回到mysql。今天终于有点时间先在内网实践一翻。

## 运行环境
Ubuntu 20.04

## 操作过程
#### 安装TiCDC
使用TiDB生态工具TiCDC
方法有很多，在这里使用 TiUP 扩容 TiDB 集群。

给定两台主机作为 TiCDC 节点，官方建议最少2台。IP 地址为 10.0.0.60、10.0.0.61，可以按照如下步骤进行操作。

1. 添加节点信息到 scale-out.yaml 文件
编写 scale-out.yaml 文件：

```
cdc_servers:
  - host: 10.0.0.60
  - host: 10.0.0.61
```

2. 运行扩容命令
```
tiup cluser scale-out xxx-cluster scale-out.yaml --user xxx-deploy -i /home/xxx-deploy/.ssh/id_rsa
```

**注意：** 官方说辞，此处假设当前执行命令的用户和新增的机器打通了互信，如果不满足已打通互信的条件，需要通过 -p 来输入新机器的密码，或通过 -i 指定私钥文件。

可是不管我是用-p还是-i都报错：
```
Error: Failed to initialize TiDB environment on remote host '10.0.0.60' (task.env_init.failed)
      caused by: Failed to create '~/.ssh' directory for user 'xxx-deploy'
        caused by: Failed to execute command over SSH for 'xxx-deploy@10.0.0.60'
          caused by: ssh: handshake failed: ssh: unable to authenticate, attempted methods [publickey none], no supported methods remain
```
#### 官方论坛帖子有分析原因：
- 1.ssh目录权限不正确
- 2.authorized_keys文件权限
- 3.使用的用户没有 sudo 权限
#### 解决方案
- .ssh 目录权限 744，authorized_keys 文件权限 600
- 用户添加 sudo 权限
- 确定免密配置好。使用密钥的，将中控机密钥 copy 到远程服务器

似乎就是这个问题，但是由于之前内网的集群不是我配置的，tiup工具在root，然后配置文件user是用xxx-deploy用户集群里面又有tidb用户，有点混乱。我已经对照解决方案逐一检查过了还是不可以。

折磨了一下午后，终于解决了，就是sudo权限问题。登陆到远程服务器之后执行 sudo su - 命令，需要输入密码。sudo配置需要配置成 tidb ALL=(ALL) NOPASSWD:ALL 之后就可以了。
我查看了原有集群的/etc/sudoers文件，照样修改了2台新机器的/etc/sudoers文件。
```
sudo su -
chmod a+w /etc/sudoers
vim /etc/sudoers
chmod a-w /etc/sudoers
```
修改后文件如下：
```
# 省略其他代码
# User privilege specification
root    ALL=(ALL:ALL) ALL
xxx-deploy ALL=(ALL) NOPASSWD:ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
# 省略其他代码
```

再次执行：
```
tiup cluster scale-out xxx-cluster scale-out.yaml --user xxx-deploy -i /home/xxx-deploy/.ssh/id_rsa
```
终于看到激动人心的Scaled cluster `xxx-cluster` out successfully字样。

### 使用tiup创建 cdc 同步任务