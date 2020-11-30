### 简介

分布式锁这个东西做后端的肯定会用到的，最近正遇到了一些棘手的问题，正好可以用这个解决，因为系统已经部署了一套 Redis 环境，就不想引入 zookeeper 那样专业的分布式组件了。

具体问题是一个很常见的业务场景，比如商城订单，一般都会有超时未支付自动取消这样的逻辑，PHP 系统一般借助于 Crontab 调度每分钟跑一次脚本去实现。

但是 Crontab 默认是上一分钟不管任务有没有结束，到了下一分钟任务又执行了。这样对于数据量大，包含调用第三方接口的话，很有可能一分钟跑不完，那没跑完的那批订单下次又会被跑一遍，可能会出现意料之外的问题。于是考虑 Redis 分布式锁。

###思路

要解决的问题很简单，但是实现起来却是有很多细节。基本思路就是用 setnx 的思想去做：

##### 原子性

加锁，释放，我们都需要封装成原子操作。加锁我们可以直接用 set 命令做，```set k v EX 60 NX``` 这样的形式可以直接实现加锁，因为是一条命令完成，本身就是原子性的；释放的话，我们必须检测当前获取到的值是不是当前进程生成的值，以防止释放了别的进程的锁，所以这个释放操作必须通过 lua 脚本实现原子性。

##### 续期

续期是必须要有的，考虑到进程开始获取到了锁，但是执行时间太长，会导致锁过期，别的进程就能获取到锁了，明显不符合期望。所以我们必须有一个进程去不停的”询问“，”请问你完成任务没有？“，没有完成的话，重置一下锁的 ttl。

### 实现

RedisLock.php:

```php
<?php
/**
 * Created by
 * Author purelight
 * Date 2020-11-28
 * Time 00:43
 */


namespace Purelightme\RedisLock;


use Predis\Client;
use Predis\Response\Status;

/**
 * Redis分布式锁
 *
 * Class RedisLock
 */
class RedisLock
{
    /**
     * @var Client
     */
    protected $client;

    /**
     * @var string key
     */
    protected $name = 'default';

    /**
     * @var string requestId
     */
    protected $requestId;

    /**
     * @var int expire seconds
     */
    protected $ttl = 60;

    public function __construct(array $params)
    {
        if (isset($params['name'])) {
            $this->name = $params['name'];
        }
        if (isset($params['ttl'])) {
            $this->ttl = $params['ttl'];
        }
        $this->requestId = uniqid($this->name, true);
        unset($params['name'],$params['ttl']);
        $this->client = new Client($params);
    }

    /**
     * Attempt to acquire the lock
     *
     * @return bool
     */
    public function acquire()
    {
        $res = $this->client->set($this->name, $this->requestId, 'EX', $this->ttl, 'NX');
        if ($res instanceof Status){
            return $res->getPayload() === 'OK';
        }
        return false;
    }

    /**
     * Reset the lock ttl
     *
     * @return bool
     */
    public function keepAlive()
    {
        $lua = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('expire', KEYS[1], " .
            $this->ttl . ") else return 0 end";
        return $this->client->eval($lua, 1, $this->name, $this->requestId) === 1;
    }

    /**
     * Release the lock
     *
     * @return bool
     */
    public function release()
    {
        $lua = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        return $this->client->eval($lua, 1, $this->name, $this->requestId) === 1;
    }

    public function __toString()
    {
        return $this->name.':'.$this->requestId.'['.$this->ttl.']';
    }
}
```

SequenceTask.php

```php
<?php
/**
 * Created by
 * Author purelight
 * Date 2020-11-28
 * Time 10:47
 */


namespace Purelightme\RedisLock;


use Purelightme\RedisLock\Exception\InternalException;
use Throwable;

class SequenceTask
{
    public static function execute(array $config, $callable)
    {
        $lock = new RedisLock($config);
        if ($lock->acquire() === false) {
            throw new InternalException('获取锁失败:'.$lock);
        }

        $pid = pcntl_fork();
        if ($pid === -1) {
            throw new InternalException('子进程创建失败');
        } elseif ($pid === 0) {
            //子进程
            for ($i = 0; ; $i++) {
                $interval = $config['interval'] ?? $config['seconds'] ?? 40;
                sleep($interval);
                if ($lock->keepAlive() === false) {
                    exit(0);
                }
            }
        } else {
            //父进程
            try{
                $res = $callable();
            }catch (Throwable $exception){
                $res = $exception;
            }finally{
                if ($lock->release() === false){
                    throw new InternalException('释放锁失败:'.$lock);
                }
            }
            pcntl_waitpid($pid,$status);
            if ($res instanceof Throwable){
                throw $res;
            }
            return $res;
        }
    }
}
```

测试串行任务：

```php
<?php
/**
 * Created by
 * Author purelight
 * Date 2020-11-28
 * Time 11:13
 */

use Purelightme\RedisLock\Exception\InternalException;
use Purelightme\RedisLock\SequenceTask;

require_once __DIR__ . '/../vendor/autoload.php';

$config = [
    'host' => 'redis',
    'name' => 'default',    //Redis key名称
    'ttl' => 60,            //Redis Key的ttl
    'interval' => 5,        //子进程续期的时间间隔
];

try{
    $res = SequenceTask::execute($config,function (){
        //fake long time logic...
        sleep(20);
//        throw new Exception('业务逻辑出错');
        return 'job execute success';
    });
}catch (InternalException $exception){
    //redis-lock 内部异常
    $res = $exception->getMessage();
}catch (Throwable $exception){
    //业务逻辑代码异常,看情况处理
    $res = $exception->getMessage();
}

var_dump($res);
```

这样，我们的业务代码只需要封装在 callable 里面，SequenceTask 将会自动保证该任务的分布式串行。

### 总结

该 package 已发布至 packagist ，可直接通过 composer 安装使用：[github源码](https://github.com/Purelightme/redis-lock)

```
composer require purelightme/redis-lock
```

因为主要是针对解决任务调度冲突的问题，所以暂不考虑 "可重入"，这种业务场景也不需要重入。



```2020-11-30```

