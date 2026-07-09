---
title: redis基础
published: 2026-07-01
description: redis的基础使用及其基本配置和命令
image: " "
tags: [linux,redis,docker]
category: 学习笔记
draft: false
---

于debian12下，docker容器化环境

**安装下载配置这些就不演示了**
**简单说一下，就是在docker里下载镜像启动容器对接端口...**

# 启动
```
redis-server
```

# 登陆命令相关：

登陆cli：
```
redis-cli <-h (指定服务器 IP)> <-p (指定端口号)>
```

原始形式显示: (中文显示)    
```
-raw
在登陆命令上：redis-cli --raw
```

# 通用key命令:
key查找：
```
keys <  > 
```

检查给定的 key 的存在:
```
exists <key>  (存在返回 1，否则返回 0)
```

修改 key 的名称:
```
rename <key> <newkey>
```

删除key：
```
DEL <keys> 
```

设置过期时间：
```
expire <key> <time> （默认为秒）
```

查看过期时间：
```
ttl <key>
```

删除所以的键值对：
```
flushall
```

# String (最基本的数据类型:字符串)
设定键值对：
```
set <key> <value>
```

当key不存在时 设定键值对：
```
setnx <key> <value>
```

查看key对应的列表(值) 范围的元素：
```
LRANGE <key> <start> <stop>  
例：LRANGE uwu 0 -1	(查看全部的元素)
```

# List(列表)
**key对应的value可以是装载多个元素的:List(列表)**

插入添加列表头部：
```
LPUSH <key> <value>
```

插入添加列表尾部：
```
RPUSH <key> <value>
``` 

从删除头部的元素:
```
LPOP <key> <count> (默认一个)
```

从删除尾部的元素:
```
RPOP <key> <count> (默认一个)
```

查看列表长度：
```
LLEN <key>
```

只保留范围元素，其他全部删除:
```
LTRIM <key> <start> <stop>
```

# Set (集合)
**key对应的value可以是有去重性,和无序性的:Set (集合)**

集合：
```
SADD <key> <members>
```

查看集合：
```
SMEMBERS <key>
```

查看元素是否在集合:
```
SISMEMBER <key> <member>
```

删除元素：
```
SREM <key> <member>
```

# 有序集合
**key对应的有去重性集合里的元素(member) 又有其对应的数字(score)**

有序集合添加：
```
ZADD <key> <score member>
```

查询集合元素：
```
ZRANGE <key> <start> <stop> [WITHSCORES](同时输出对应元素的值)
```

查看对应元素的值:
```
ZSCORE <key> <member>
```

查看对应元素在集合的排名(从小到大)：
```
ZRANK <key> <member>
```

查看对应元素在集合的排名(从大到小)：
```
ZREVRANK <key> <member>
```

移除有序集合中的元素：
```
ZREM <key> <member>
```

# Hash哈希(键值对集合)
**key对应的哈希表里的字段(field) 又有其对应的值(value)**

在哈希表添加键值对：
```
HSET <key> <field> <value>
```

获取表中元素对应的值：
```
HGET <key> <field>
```

获取表中多个元素对应的值:
```
HMGET <key> <fieldS> 
```

获取表内所以的键值对:
```
HGETALL <key>
HKEYS <key>
```

删除表内元素:
```
HDEL <key> <field>
```

检查存在：
```
HEXISTS <key> <field>
```

表内键值对数:
```
HLEN <key>
```

# 频道

发布：
```
publish <channel> <message>
```

订阅：
```
subscribe <channel> <message>
```

# 事务：
**将多个命令打包成一个整体，按顺序一次性执行，不会因为某一个命令报错而中断后续命令**
```
MULTI 	进入事务模式
EXEC	退出事务模式
```

# 备份
备份命令
```
前台备份：
save
后台备份：
bgsave
```

# 配置文件修改
**redis目录下的reids.conf**
```
bind 连接的主机ip
port 端口
pidfile 进程id
dbfilename 备份文件名称
replicaof 绑定主节点
appendonly 持久化
```

# 主从复制
**先将redis配置文件复制一份**
需要修改的配置：
```
port 端口
pidfile 进程id
dbfilename 备份文件名称
replicaof 绑定主节点
```
然后在启动从属:
```
redis-server <配置文件路径和名称> <对应从属机的redis端口>
```

# 哨兵模式
**哨兵模式在主从复制下完成的，需要先进行主从复习**
先配置哨兵配置文件：
**新建sentinel.conf，在文件里配置**:
```
sentinel monitor <master name> <master ip> <master port> <同意故障转移需要的数量>
```
然后启动
```
redis-sentinel sentinel.conf
```