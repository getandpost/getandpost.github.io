## MySQL之binlog日志恢复数据库

今天回来被告知公司的一个项目被人删库了。仔细查看发现是被勒索btccoin了，对方留有勒索信云云。

作为一个历经沧桑的程序员遇到这种事情，，也是第一次啊！！但我们也不能慌，第一时间查看了主从复制的从库发现也是空空如此，删除数据库的操作也同步过去了。

既然我有做主从那必然是开启了binlog的，于是查看/usr/local/mysql/var，哦耶~mysql-bin日志等文件都还在，这下赎金应该可以省下了。于是尝试着利用binlog恢复数据库。

由于笔者也没搞过，一时间竟然无从下手。思想挣扎如下：
- 先是想着直接在服务器上做恢复到原库。但又怕恢复失败把数据都弄没了。
- 还是把tar然后scp到本地服务器上搞吧。
- 整个/usr/local/mysql/var日录打包下来后在想是要替换本地服务器的/usr/local/mysql/var目录去恢复到原库：原因是之前发生过服务器烧坏，硬盘曾拿去数据修复。然后当时负责人把整个/var文件拷进去做恢复。虽然我有全程参与但时隔多年，细节早全忘了。

按国际惯例，利用搜索引擎查阅大量有关资料，对利用mysqlbinlog恢复的初步印象如下：
- 直接恢复到数据库：执行命令/usr/bin/mysqlbinlog --start-datetime="2020-08-12 14:28:15" --stop-datetime="2020-09-28 18:56:27" --database=db_name /var/lib/mysql/mysql-bin.000009 | /usr/bin/mysql -uroot -pxxx -v db_name
- 先读取日志到sql文件：mysqlbinlog --no-defaults -d DB_name --start-datetime='2020-08-12 14:28:15' --stop-datetime='2020-09-28 18:56:27' mysql-bin.000032 > backup.sql

无论哪种都很疑惑，因数binlog日志那么多个。然后去定义开始、结束时间。或者开始、结束点。是一条命令把所有日志都写上呢？还是可以不用写，或者加个|more？然后，又是怎么样找开始、结束的点呢？

带着这些疑惑，我中午都没休息，顶着困意去磕碰去摸索。

最后思路逐渐清晰起来：所谓利用binlog日志恢复数据操作记录其实就是定时全备份+binlog日志恢复增量数据部分。幸好之前还留有1个多月前的全量备份。再次体现定时备份的重要性。

```
# 查找开始的点：
mysqlbinlog mysql-bin.000051; 
# at 434045590
#200812 14:31:16 server id 1  end_log_pos 434045621 CRC32 0x069b23e4    Xid = 13636328

# 给每个mysql-bin日志切割出sql文件： 
mysqlbinlog --start-position="434045590" --database=db_name mysql-bin.000051 > /home/db_name/db_name.backup-000051-0812.sql;
mysqlbinlog --database=db_name mysql-bin.000052 > /home/db_name/db_name.backup-000052.sql;
mysqlbinlog --database=db_name mysql-bin.000053 > /home/db_name/db_name.backup-000053.sql;
mysqlbinlog --database=db_name mysql-bin.000054 > /home/db_name/db_name.backup-000054.sql;
mysqlbinlog --database=db_name mysql-bin.000055 > /home/db_name/db_name.backup-000055.sql;
mysqlbinlog --stop-position="8165734" --database=db_name mysql-bin.000056 > /home/db_name/db_name.backup-000056-8165734.sql;

# 利用source命令导入sql
mysql -uroot -p;
use db_name;
source /home/db_name/db_name_20200812.sql;
source /home/db_name/db_name.backup-000051-0812.sql;
```

mysqlbinlog常见的选项的以及其他用法后续补充。

至此终于把数据库给恢复过来了，化险为夷！

