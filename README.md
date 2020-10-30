# Redis-Notes

Practice redis setting by using docker and kubernetes

version:
* redis server:
  - redis:5.0.6-alpine
* test client:
  - python:3.7.4-alpine


## reference

* [https://redis.io/](https://redis.io/)
* [download documentation](http://download.redis.io/redis-stable/)
* [redis.conf](http://download.redis.io/redis-stable/redis.conf)
* [sentinel.conf](http://download.redis.io/redis-stable/sentinel.conf)

# basic

## docker
0. 啟動 & 關閉:
  ```bash
  cd basic
  docker-compose up -d  # 啟動
  docker-compose down  # 關閉
  ```
1. configuration 配置透過 redis.conf 配置，並且在啟動時餵給 redis-server
2. persistence 會在機器內的 /data 下
3. 配置也可以透過參數直接給 ex.
  ```
  redis-server --port 6380
  ```
4. client 原生指令 redis-cli
  ```bash
  docker exec -it redis redis-cli INFO
  ```

# master-slave

## docker
* 查看 replication 狀態
  ```bash
  docker exec redis-master redis-cli INFO replication
  docker exec redis-slave1 redis-cli INFO replication
  docker exec redis-slave2 redis-cli INFO replication
  ```
* 讀寫測試
  ```bash
  docker exec redis-master redis-cli SET key value
  # OK
  docker exec redis-master redis-cli GET key
  # value
  docker exec redis-slave1 redis-cli GET key
  # value
  docker exec redis-slave2 redis-cli GET key
  # value
  docker exec redis-slave2 redis-cli set key2 value2
  # READONLY You can't write against a read only replica.
  ```

# [High Availability](https://redis.io/topics/sentinel)

redis 實作 HA 的機制為 sentinel，利用 sentinel 來監控 master 服務是否安好，

若服務掛點， sentinel 有幾種方式可以來達到高可用，以下實作文檔中 [basic setup with three boxes](https://redis.io/topics/sentinel#example-2-basic-setup-with-three-boxes) 的情況

## config

```
# sentinel monitor <master-group-name> <ip> <port> <quorum>
# quorum: The quorum is the number of Sentinels that need to agree about the fact the master is not reachable,
# in order to really mark the master as failing, and eventually start a failover procedure if possible.
sentinel monitor mymaster redis-master 6379 2
# sentinel <option_name> <master_name> <option_value>
# down-after-milliseconds: is the time in milliseconds an instance should not be reachable 
# (either does not reply to our PINGs or it is replying with an error) for a Sentinel starting to think it is down.
sentinel down-after-milliseconds mymaster 5000
# sentinel 執行 failover 失敗時間為 60000 毫秒
sentinel failover-timeout mymaster 60000
# sets the number of replicas that can be reconfigured to use the new master after a failover at the same time. 
# The lower the number, the more time it will take for the failover process to complete, 
# however if the replicas are configured to serve old data, 
# you may not want all the replicas to re-synchronize with the master at the same time. 
# While the replication process is mostly non blocking for a replica, 
# there is a moment when it stops to load the bulk data from the master. 
# You may want to make sure only one replica at a time is not reachable by setting this option to the value of 1.
sentinel parallel-syncs mymaster 1
```

## docker
* 啟動 sentinel 可以用 redis-sentinel 或是 redsi-server:
  ```bash
  redis-sentinel path/to/sentinel.conf
  redis-server path/to/sentinel.conf --sentinel
  ```
* 查看 sentinel 狀態
  ```bash
  docker exec sentinel1 redis-cli -p 26379 INFO sentinel
  docker exec sentinel2 redis-cli -p 26379 INFO sentinel
  docker exec sentinel3 redis-cli -p 26379 INFO sentinel
  ```
* 查找 master
  ```bash
  docker exec sentinel1 redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
  ```
* master 轉移測試
  ```bash
  # 把 master 停掉, ps. redis-cli -p 6379 DEBUG sleep 30 也可以
  docker-compose stop master
  # 查看兩個 slave 狀態, 是不是有一個變成 master
  docker exec replica1 -a pass1234 redis-cli INFO replication
  docker exec replica2 -a pass1234 redis-cli INFO replication
  # 查找 master , 是不是換了新 ip
  docker exec sentinel1 redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
  # master 重新啟動
  docker-compose start master
  # 在查看 master 狀態，變成 slave
  docker exec master redis-cli -a pass1234 INFO replication
  ```

# [Cluster](https://redis.io/topics/cluster-tutorial)

## docker
* create redis cluster (版本 5 以上才支援，4 以下要去載 ruby client)
  ```bash
  docker exec -it redis-node1 \
    redis-cli --cluster create 172.19.0.2:6379 172.19.0.3:6379 \
    172.19.0.4:6379 172.19.0.5:6379 172.19.0.6:6379 172.19.0.7:6379 \
    --cluster-replicas 1
  # yes
  # ...
  # [OK] All nodes agree about slots configuration.
  # >>> Check for open slots...
  # >>> Check slots coverage...
  # [OK] All 16384 slots covered.
  ```
* node 狀態
  ```bash
  docker exec redis-node1 redis-cli cluster nodes
  ```
* 讀寫
  ```bash
  docker exec redis-node1 redis-cli -c SET k1 v1
  docker exec redis-node2 redis-cli -c SET k2 v2
  docker exec redis-node3 redis-cli -c SET k3 v3
  docker exec redis-node4 redis-cli -c SET k4 v4
  docker exec redis-node5 redis-cli -c GET k2
  docker exec redis-node6 redis-cli -c GET k3
  ```

to be continued...