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