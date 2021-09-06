# hyperf/socker.io重启出现异常和解决
```
描述：新安装hyperf/socketio-server重启出现协程异常
```

## 安装

composer install -o

配置redis（使用房间）

启动：docker-compose up -d

重启：docker-compose restart
```
docker logs -f hyperf

[INFO] Worker#1 started.
[INFO] Worker#3 started.
[INFO] Worker#2 started.
[INFO] Worker#0 started.
[INFO] WebSocket Server listening at 0.0.0.0:9502
[INFO] HTTP Server listening at 0.0.0.0:9501
[2021-09-06 06:15:49 #1.4]      INFO    Server is shutdown now
[2021-09-06 06:15:52 *20.0]     WARNING Worker_reactor_try_to_exit() (ERRNO 9012): worker exit timeout, forced termination

===================================================================
 [FATAL ERROR]: all coroutines (count: 1) are asleep - deadlock!
===================================================================

 [Coroutine-3]
--------------------------------------------------------------------
#0  Redis->zRangeByScore() called at [/data/hyperf/vendor/hyperf/redis/src/RedisConnection.php:76]
#1  Hyperf\Redis\RedisConnection->__call() called at [/data/hyperf/vendor/hyperf/redis/src/Redis.php:49]
#2  Hyperf\Redis\Redis->__call() called at [/data/hyperf/vendor/hyperf/redis/src/RedisProxy.php:32]
#3  Hyperf\Redis\RedisProxy->__call() called at [/data/hyperf/vendor/hyperf/socketio-server/src/Room/RedisAdapter.php:207]
#4  Hyperf\SocketIOServer\Room\RedisAdapter->cleanUpExpiredOnce() called at [/data/hyperf/vendor/hyperf/socketio-server/src/Room/RedisAdapter.php:199]
#5  Hyperf\SocketIOServer\Room\RedisAdapter->Hyperf\SocketIOServer\Room\{closure}() called at [/data/hyperf/vendor/hyperf/utils/src/Functions.php:271]
#6  call() called at [/data/hyperf/vendor/hyperf/utils/src/Coroutine.php:62]

[INFO] Worker#1 started.
[INFO] Worker#2 started.
[INFO] Worker#3 started.
[INFO] Worker#0 started.
[INFO] WebSocket Server listening at 0.0.0.0:9502
[INFO] HTTP Server listening at 0.0.0.0:9501
```

## 异常出现位置

> ./vendor/hyperf/socketio-server/src/Room/RedisAdapter.php:198

## 异常原因

```
public function cleanUpExpired(): void
    {
        Coroutine::create(function () {
            while (true) {
                CoordinatorManager::until(Constants::WORKER_EXIT)->yield($this->cleanUpExpiredInterval / 1000);
                $this->cleanUpExpiredOnce();
            }
        });
    }

此代码对work进程退出没有做处理，导致当使用`docker-compose restart`重启时出现无法正常退出协程报错
```

## 解决方式

```
public function cleanUpExpired(): void
    {
        Coroutine::create(function () {
            while (true) {
                if (CoordinatorManager::until(Constants::WORKER_EXIT)->yield($this->cleanUpExpiredInterval / 1000)) {
                // work进程退出停止循环
                 break;
                }
                $this->cleanUpExpiredOnce();
            }
        });
    }

对进程退出做处理
```

