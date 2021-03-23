背景

1. User 模块直接查 mysql，多处被调用查询，访问量上升后容易造成性能瓶颈
2. 接口数据杂糅，融合多表的数据结果，需要整理
3. 制定缓存机制，甚至希望是缓存框架的方向，方便其他业务数据模块移植使用

### 目标
（暂定）初步目标是 10w 日活，高峰时段 18 点~ 21 点，根据二八定律，80% 的用户在高峰时段使用，firebase 接口平均时长 100ms 左右
并发量 100000 * 0.8 / (4 * 3600) = 5

单 server 并发 100，QPS 1000（假设平均请求时间 100ms）

### 优化前性能数据

#### 正式环境（海外）

firebase:  GET/PATCH user 平均 100ms

postman: 查询单个 user 140~230 ms

#### 开发环境

postman: 查询单个 user 平均 50ms

ab 压测数据：

注意保证接口参数正确，避免鉴权等原因报错直接返回，没走完接口逻辑，这样测试结果没有意义

```
get user 最大并发 10(全部成功，受限于数据库连接数限制，超过会程序报错)，QPS 163
Concurrency Level:      10
Time taken for tests:   6.128 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      1027862 bytes
HTML transferred:       472000 bytes
Requests per second:    163.19 [#/sec] (mean)
Time per request:       61.280 [ms] (mean)
Time per request:       6.128 [ms] (mean, across all concurrent requests)
Transfer rate:          163.80 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        6    8   1.2      7      17
Processing:    14   53  20.7     54     194
Waiting:       14   53  20.7     54     194
Total:         21   61  20.8     62     202

Percentage of the requests served within a certain time (ms)
  50%     62
  66%     64
  75%     65
  80%     66
  90%     70
  95%     78
  98%    127
  99%    179
 100%    202 (longest request)
```
```
get user_state 
```



### 优化点

1. 引入 redis 做缓存，缓存热点数据

   user

   user state

   黑名单（评估性能）

   friend follow（评估性能）

2. dao 接口整理，同步缓存

   get one

   get some

   create one

   update one

   get one with cols

   get some with cols

   relate_user_profiles（是 dict 合并，跟 get some with cols 不太一样，但也需要 get by ids）

   search by key word

3. get some 如果部分在缓存中不存在，则需要从 db 查询后同步到缓存

4. 调整 db redis 的连接数，并保证并发数大于连接数时程序不报错

   ```
   # mysql 最大连接数
   SHOW VARIABLES LIKE "max_connections";
   
   # redis 最大连接数
   config get maxclients
   ```

   

5. 偶尔直接改库，不考虑同步到 redis 的话，接受一小时的不一致时间

### 缓存同步策略思考

cache aside

读：先读 cache，没有再读 db，并同步回 cache（设置失效时间）

写：先写 db，再删 cache（删 cache 比更新实现简单，仅多了一次 cache miss）

### 整改

1. 增加 user redis 缓存，有效提升 QPS

```
get user 设置并发 10(保持变量)，QPS 220
Concurrency Level:      10
Time taken for tests:   4.499 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      1027904 bytes
HTML transferred:       472000 bytes
Requests per second:    222.28 [#/sec] (mean)
Time per request:       44.988 [ms] (mean)
Time per request:       4.499 [ms] (mean, across all concurrent requests)
Transfer rate:          223.13 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        6    8   1.1      8      16
Processing:    12   37   8.6     37     105
Waiting:       12   37   8.6     37     105
Total:         19   45   8.7     44     112

Percentage of the requests served within a certain time (ms)
  50%     44
  66%     46
  75%     47
  80%     47
  90%     50
  95%     54
  98%     63
  99%     99
 100%    112 (longest request)
```

```
get user 设置并发 11(部分失败，受限于连接数)，QPS 214
Concurrency Level:      11
Time taken for tests:   4.662 seconds
Complete requests:      1000
Failed requests:        3
   (Connect: 0, Receive: 0, Length: 3, Exceptions: 0)
Non-2xx responses:      3
Total transferred:      1027036 bytes
HTML transferred:       471454 bytes
Requests per second:    214.50 [#/sec] (mean)
Time per request:       51.282 [ms] (mean)
Time per request:       4.662 [ms] (mean, across all concurrent requests)
Transfer rate:          215.14 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        6    8   1.4      8      36
Processing:    18   43   7.1     42      86
Waiting:       18   43   7.1     42      86
Total:         26   51   7.4     50      94

Percentage of the requests served within a certain time (ms)
  50%     50
  66%     51
  75%     53
  80%     54
  90%     59
  95%     64
  98%     75
  99%     80
 100%     94 (longest request)
```

2. 修改 db redis 最大连接数，db 10->20，redis 10->100，避免程序报错

raise EmptyPoolError(db_name)

raise ConnectionError("Too many connections")

```
get user 设置并发 10(保持变量)，QPS 220
Concurrency Level:      10
Time taken for tests:   4.563 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      1027874 bytes
HTML transferred:       472000 bytes
Requests per second:    219.14 [#/sec] (mean)
Time per request:       45.634 [ms] (mean)
Time per request:       4.563 [ms] (mean, across all concurrent requests)
Transfer rate:          219.97 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        6    8   1.5      7      20
Processing:    11   38   6.0     37     130
Waiting:       11   37   6.0     37     130
Total:         18   45   6.2     45     139

Percentage of the requests served within a certain time (ms)
  50%     45
  66%     46
  75%     47
  80%     47
  90%     49
  95%     51
  98%     62
  99%     72
 100%    139 (longest request)
```

```
get user 设置并发 100(从 10 改到 100，对 QPS 无提升影响)，QPS 219
Concurrency Level:      100
Time taken for tests:   4.553 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      1027864 bytes
HTML transferred:       472000 bytes
Requests per second:    219.62 [#/sec] (mean)
Time per request:       455.341 [ms] (mean)
Time per request:       4.553 [ms] (mean, across all concurrent requests)
Transfer rate:          220.44 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        6   11  11.9      8      60
Processing:    29  425  76.9    442     498
Waiting:       29  424  76.9    442     497
Total:         64  436  68.5    450     506

Percentage of the requests served within a certain time (ms)
  50%    450
  66%    453
  75%    456
  80%    459
  90%    471
  95%    490
  98%    496
  99%    499
 100%    506 (longest request)
```

总结：

QPS 提升到 220 左右，同时说明提升并发量对 QPS 无影响，certain time 会等待更多时间

增大 mysql redis 连接数到 500，对 QPS 无影响，目前看来瓶颈跟连接数无关
