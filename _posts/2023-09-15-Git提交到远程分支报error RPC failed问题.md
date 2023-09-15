# Git提交到远程分支报error: RPC failed; HTTP 413 curl 22 The requested URL returned error: 413 解决方案

### 背景
工作中用使用的自建Gitlab，远程仓库克隆到自己的仓库。开发在本地提交到自己的远程，然后再Gitlab上发起merge request到远程。

### 遇到的问题
由于我一段时间没有push到自己的仓库了，大概有半年。落后了好多。当我想推到自己仓库对应分支时报错error: RPC failed; HTTP 413 curl 22 The requested URL returned error: 413：
```
$ git push origin dev
Enumerating objects: 13846, done.
Counting objects: 100% (12756/12756), done.
Delta compression using up to 6 threads
Compressing objects: 100% (6460/6460), done.
error: RPC failed; HTTP 413 curl 22 The requested URL returned error: 413
fatal: the remote end hung up unexpectedly
Writing objects: 100% (12282/12282), 55.82 MiB | 14.89 MiB/s, done.
Total 12282 (delta 9236), reused 7920 (delta 5750), pack-reused 0
fatal: the remote end hung up unexpectedly
Everything up-to-date
```
网上搜索一翻

### 原因
HTTP上传文件限制了文件大小

此时改变git配置是没有用的，因为这跟本不是git的问题

### 解决方案
网上的解决方案是更换远程仓库地址，比如换SSH git remote set-url origin 复制的地址
```
git remote set-url origin sftp://username@your_server/path_to_repo
```
试了一下发现没有用。

既然原因是太大了，于是我想到了一个解决办法：分阶段提交。具体操作如下：
先reset到某个commit，然后分段提交。
```
git log
git reset --hard commit id
git push origin dev
```
积累了6个月的提交记录，大概分个6次就可以了。