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