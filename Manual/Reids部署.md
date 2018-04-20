# Redis安装与配置

记录安装与配置redis的过程，方便查询。

# 1. 安装

下载redis：[https://redis.io/download](https://redis.io/download)

这里下载稳定版 `redis-4.0.9.tar.gz` ，下载目录为 `/home/veee` ，安装目录为 `/usr/local/redis` 。

```shell
> cd /home/veee
> tar -zxvf redis-4.0.9.tar.gz  # 解压
> mv redis-4.0.9 /usr/local/  # 移动
> cd /usr/local
> mv redis-4.0.9 redis  # 重命名
> cd redis
> make && make install  # 编译
```

# 2. 配置

这里提供部署redis的sentinel模式所需的参考配置。

配置文件所在目录： `/usr/local/redis/conf-file/` 。

- master配置 `redis-replication-conf/redis-master-6379.conf` :

  ```
  bind 192.168.66.151
  protected-mode no
  port 6379
  daemonize yes
  requirepass "123456"
  pidfile "/var/run/redis_6379.pid"
  ```

- slave_1配置 `redis-replication-conf/redis-slave-6380.conf` ：

  ```
  bind 192.168.66.151
  protected-mode no
  port 6380
  daemonize yes
  requirepass "123456"
  pidfile "/var/run/redis_6380.pid"

  slaveof 192.168.66.151 6379
  ```

- slave_2配置 `redis-replication-conf/redis-slave-6381.conf` ：

  ```
  bind 192.168.66.151
  protected-mode no
  port 6381
  daemonize yes
  requirepass "123456"
  pidfile "/var/run/redis_6381.pid"

  slaveof 192.168.66.151 6379
  ```

- sentinel_1配置 `redis-sentinel-conf/redis-sentinel-5000.conf` ：

  ```
  bind 192.168.66.151
  port 5000
  protected-mode no
  daemonize yes

  sentinel myid 9617576842794bcad3dac0fb039901cac0b67639
  sentinel monitor mymaster 192.168.66.151 6379 2
  sentinel down-after-milliseconds mymaster 5000
  sentinel failover-timeout mymaster 60000
  sentinel auth-pass mymaster 123456
  ```

- sentinel_2配置 `redis-sentinel-conf/redis-sentinel-5001.conf` ：

  ```
  bind 192.168.66.151
  port 5001
  protected-mode no
  daemonize yes

  sentinel myid 9617576842794bcad3dac0fb039901cac0b67639
  sentinel monitor mymaster 192.168.66.151 6379 2
  sentinel down-after-milliseconds mymaster 5000
  sentinel failover-timeout mymaster 60000
  sentinel auth-pass mymaster 123456
  ```

- sentinel_3配置 `redis-sentinel-conf/redis-sentinel-5002.conf` ：

  ```
  bind 192.168.66.151
  port 5002
  protected-mode no
  daemonize yes

  sentinel myid 9617576842794bcad3dac0fb039901cac0b67639
  sentinel monitor mymaster 192.168.66.151 6379 2
  sentinel down-after-milliseconds mymaster 5000
  sentinel failover-timeout mymaster 60000
  sentinel auth-pass mymaster 123456
  ```

# 3. 启动

首先启动主从：

```shell
> cd /usr/local/redis
> src/redis-server conf-file/redis-replication-conf/redis-master-6379.conf
> src/redis-server conf-file/redis-replication-conf/redis-slave-6380.conf
> src/redis-server conf-file/redis-replication-conf/redis-slave-6381.conf
```

然后启动哨兵：

```shell
> cd /usr/local/redis
> src/redis-sentinel conf-file/redis-sentinel-conf/redis-sentinel-5000.conf
> src/redis-sentinel conf-file/redis-sentinel-conf/redis-sentinel-5001.conf
> src/redis-sentinel conf-file/redis-sentinel-conf/redis-sentinel-5002.conf
```

# 4. 查看运行信息

*待补充……*

