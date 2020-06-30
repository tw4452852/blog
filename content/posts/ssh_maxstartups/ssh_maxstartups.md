+++
title = "ssh MaxStartups"
date = "2016-09-20T15:00:00"
tags = [ "linux", "concurrency" ]
+++

## background

这几天用go写了个小工具 , 主要为了方便在项目更新代码 . 
由于项目中有多个git仓库 , 为了快速更新 , 使用了多线程同时更新
不同的仓库 , 但是经常有些仓库代码更新失败 . 提示的错误如下 :

```
ssh_exchange_identification: Connection closed by remote host
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.

ssh_exchange_identification: read: Connection reset by peer
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

经过google一番 , 也有人遇到同样的问题 , 而且该问题只出现在多线程
并发情况下 , 根本原因出在sshd的MaxStartups选项上 . 

## MaxStartups

首先来看下 `MaxStartups` 的说明 :

```
MaxStartups
     Specifies the maximum number of concurrent unauthenticated connections to the SSH
     daemon.  Additional connections will be dropped until authentication succeeds or
     the LoginGraceTime expires for a connection.  The default is 10:30:100.

     Alternatively, random early drop can be enabled by specifying the three colon sepa‐
     rated values “start:rate:full” (e.g. "10:30:60").  sshd(8) will refuse connection
     attempts with a probability of “rate/100” (30%) if there are currently “start” (10)
     unauthenticated connections.  The probability increases linearly and all connection
     attempts are refused if the number of unauthenticated connections reaches “full”
     (60).
```

意思就是说 , 当未认证的连接数达到一定的阈值 , 之后来的新的连接将有选择的
丢弃 , 当未认证的连接数还在递增达到了更高的阈值 , 那么之后来的连接将直接
丢弃 , 该配置就是用来配置这两个阈值 ( start和full ) 以及可选择丢弃的丢弃
率 ( rate ) . 

查看服务端sshd的配置 , 果然 , 它使用的是默认值（10:30:100 ), 将其start和
full值改大 :

```
MaxStartups 100:30:200
```

重启sshd , 问题解决 !

FIN.
