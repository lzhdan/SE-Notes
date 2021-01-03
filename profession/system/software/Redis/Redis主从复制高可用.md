# Redis 主从复制 + 高可用
***
## 简介
&ensp;&ensp;&ensp;&ensp;使用docker配置Redis主从复制，配置 sentinel 实现高可用，一主两从

## 配置记录
### Redis主从复制
&ensp;&ensp;&ensp;&ensp;准备配置文件，分别启动启动在6380/6381/6382,6380为主

```shell
port 6379
bind 192.168.11.202
requirepass "myredis"
daemonize yes
logfile "6379.log"
dbfilename "dump-6379.rdb"
dir "/opt/soft/redis/data"
```

```shell
docker run -dit --name redis1 -p 6381:6379 redis 
docker run -dit --name redis2 -p 6382:6379 redis 
docker run -dit --name redis3 -p 6383:6379 redis 


docker exec -ti redis2 redis-cli

127.0.0.1:6379> REPLICAOF 192.168.101.104 6381
OK

127.0.0.1:6379> info replication
# Replication
role:slave
master_host:192.168.101.104
master_port:6381
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_repl_offset:126
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:3003c01fb5c96132dbc1e366cdc94b214a0ac52d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:126
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:126


docker exec -ti redis3 redis-cli

127.0.0.1:6379> REPLICAOF 192.168.101.104 6381
OK

127.0.0.1:6379> info replication
# Replication
role:slave
master_host:192.168.101.104
master_port:6381
master_link_status:up
master_last_io_seconds_ago:7
master_sync_in_progress:0
slave_repl_offset:98
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:3003c01fb5c96132dbc1e366cdc94b214a0ac52d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:98
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:71
repl_backlog_histlen:28


docker exec -ti redis1 redis-cli

127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=172.17.0.1,port=6379,state=online,offset=140,lag=0
slave1:ip=172.17.0.1,port=6379,state=online,offset=140,lag=0
master_replid:3003c01fb5c96132dbc1e366cdc94b214a0ac52d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:140
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:140
```


### 配置 Sentinel 哨兵
```shell
docker cp entinel.conf redis1:/etc/setinel.conf

docker exec -ti redis1 redis-sentinel /etc/setinel.conf

44:X 03 Jan 2021 06:39:16.506 # +monitor master mymaster 192.168.101.104 6381 quorum 2
44:X 03 Jan 2021 06:39:16.512 * +slave slave 172.17.0.1:6379 172.17.0.1 6379 @ mymaster 192.168.101.104 6381
44:X 03 Jan 2021 06:39:26.666 * +convert-to-slave slave 172.17.0.1:6379 172.17.0.1 6379 @ mymaster 192.168.101.104 6381
```