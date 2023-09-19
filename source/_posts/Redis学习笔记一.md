---
title: Redis学习笔记（一）
date: 2023-09-19 23:02:02
tags: Redis
---
# redis基础操作

## 位图操作

位图不是真正的数据,他是定义在字符串类型中。一个字符串的key最多存储512字节的内容，位上限：2^32。

## 相关命令

### setbit

设置某位置上的二进制，语法如下：

```redis
setbit key offset(偏移量) value(值)
```

效果如下：

```
set mykey ab
get mykey  ->  ab
setbit mykey 6 1
get mkey  -> cb
```

如果当前设置的位置不存在，那么redis会自动帮助我们扩展位置（默认补0）：

```
set mykey ab
get mykey  ->  ab
setbit mykey 17 1
get mkey  -> cb@
```

如果设置的key不存在，redis会自动帮我们创建这样一个key

```
setbit hello 4 1
get hello -> /b
```

### getbit

获取某一个位置上的值：

```
getbit key offset
```

### bitcount

统计键中对应的值有多少个1：

```
bitcount key start end (start/end代表索引`字节`)
```

效果如下：

```
set mykey ab
get mykey -> ab
bitcount mykey 0 0 -> 3
```

## python操作位图

```python
import redis
r = redis.Redis(host='xxxx',port='6379',db=0)
r.setbit('mykey',4,1)
print(r.getbit('mykey',3))
```

## 主从复制

它通常是指，通高可用 - 是系统架构设计中必须考虑的因素之一，过设计减少系统不能提供服务的时间目标:消除基础架构中的单点故障redis单进程单线程模式，如果redis进程挂掉，相关依赖的服务就难以正常服务。  

redis高可用方案-主从搭建 +哨兵  

1、一个Redis服务可以有多个该服务的复制品，这个Redis服务成为master，其他复制品成为slaves

2、master会一直将自己的数据更新同步给slaves，保持主从同步

3、只有master可以执行写命令，slave只能执行读命令

作用:分担了读的压力 (高并发);提高可用性原理: 从服务器执行客户端发送的读命令，客户端可以连接slaves执行读请求，来降低master的读压力

命令：

```
redis-server --slaveof <master-ip> <master-port>--masterauth <master password>
```

实际操作如下：

```
redis-serevr --slaveof 127.0.0.1 6379 --port 6300
# 客户端
redis-cli -p 6300
keys *
ser mykey abc # 此处代码错误，从的redis只读
```

此时有两条命令：
1、slaveofIP PORT 成为谁的从

2、slaveof no one自封为王

加载配置文件：

我们可以复制一个配置文件，然后在启动一个新的rendis服务的时候运用新建的配置文件。

```
6379 -> /etc/redis/redis.conf
6300 -> /home/tarean/redis_6300.conf
```

修改配置文件内容

```
vim redis_6300.conf
# 修改内容如下：
slaveof 127.0.0.1 6379 # 跟随哪个主redis
port 6300 # 启动的端口
# 修改完配置文件内容，启动新的redis服务
redis-server ./redis_6300.conf
```

## 哨兵

概念：

1、Sentinel会不断检查Master和Slaves是否正常
2、每一个Sentinel可以监控任意多个Master和该Master下的Slaves

原理:

正如其名，哨兵进程定期与 redis主从进行通信，当哨兵认为redis主阵亡后[通信无返回]，自动将切换工作完成

功能：

自动化处理`redis主从切换`

### 准备测试环境

共3个redis的服务

1、启动6379的redis服务器

```
sudo /etc/init.d/redis-server start
```

2、启动6380的redis服务器，设置为6379的从

```
redis-server --port 6380
tarena@tedu:~$ redis-cli -p 6380
127.0.0.1:6380> slaveof 127.0.0.1 6379
OK
```

3、启动6381的redis服务器，设置为6379的从

```
redis-server --port 6381
tarena@tedu:~$ redis-cli -p 6381
127.0.0.1:6381> slaveof 127.0.0.1 6379
OK
```

### 安装哨兵和启动哨兵

1、安装redis-sentinel

```
sudo apt install redis-sentinel
```

验证: `sudo /etc/init.d/redis-sentinel stop`

2、新建配置文件 sentinel.conf

```
port 26379
sentinel monitor tedu 127.0.0.1 6379 1 （如果投票超过一票那么代表此服务出现问题）
```

启动哨兵的数量最好为奇数，哨兵判断Master是否阵亡需要投票。

 3、启动sentinel
方式一: `redis-sentinel sentinel.conf`

方式二: `redis-server sentinel.conf --sentinel`

4、将master的redis服务终止，查看从是否会提升为主

```
sudo /etc/init.d/redis-server stop
```

发现提升6381为master，其他两个为从

在6381上设置新值，6380查看

```
127.0.0.1:6381> set name tedu
```

OK

完成以上的操作，哨兵会跟配置文件中的redis服务取得联系。如果Master服务挂掉了，即使后面被复活也不会重新变为Master（因为长时间没参与业务）。

### 配置文件详解

sentine1监听端口，默认是26379，可以修改

```
port 26379
```

告诉sentine1去监听地址为ip:port的一个master，这里的master-name可以自定义，quorum是一个数字，指明当有多少个sentine]认为一个naster失效时，master才算真正失效

```
sentinel monitor <master-name> <ip> <redis-port> <quorum>
```

如果master有密码，则需要添加该配置

```
sentinel auth-pass <master-name> <password>
```

master多久失联才认为是不可用了，默认是30秒

```
sentinel down-after-milTiseconds <master-name> <milTiseconds>
```

### Python操作哨兵

代码如下：

```python
from redis.sentinel import Sentinel
#生成哨兵连接
sentinel = Sentinel([('127.0.0.1',26379)],socket_timeout=0.1)
#初始化master连接
master = sentinel.master_for('tedu', socket_timeout=0.1,db=1)
slave = sentinel.slave_for('tedu',socket_timeout=0.1， db=1)
#使用redis相关命令
master.set('mymaster','yes')
print(slave.get('mymaster'))
```
