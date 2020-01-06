
案例实战：微信文章的阅读量PV
### 一、微信文章的阅读量场景介绍
这样一个场景：
在微信公众号里面的文章，每个用户阅读一遍文章，该篇文章的阅读量就会加一。如下图：
![image](https://github.com/agan-java/images/blob/master/redis/string/01.png?raw=true)  
对于微信这种一线互联网公司，如此大的并发量，一般不可能采用数据库来做计数器，通常都是用redis的incr命令来实现。

### 二、微信文章的阅读量原理:redis INCR命令
INCR命令，它的全称是increment，用途就是计数器。
每执行一次INCR命令，都将key的value自动加1。
如果key不存在，那么key的值初始化为0，然后再执行INCR操作。
例如：微信文章id=100，做阅读计算如：
```shell
127.0.0.1:6379> incr article:100
(integer) 1
127.0.0.1:6379> incr article:100
(integer) 2
127.0.0.1:6379> incr article:100
(integer) 3
127.0.0.1:6379> incr article:100
(integer) 4
127.0.0.1:6379> get article:100
"4"
```

技术方案的缺陷：
需要频繁的修改redis,耗费CPU，高并发修改redis会导致 redisCPU 100%


### 三、案例实战：编码实现微信文章的阅读量
```java
@RestController
@Slf4j
public class ViewController {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @GetMapping(value = "/view")
    public void view(Integer id) {
        //redis key
        String key="article:"+id;
        //调用redis的increment计数器命令
        long n=this.stringRedisTemplate.opsForValue().increment(key);
        log.info("key={},阅读量为{}",key, n);
    }
}
```

### 练习
这里讲的INCR命令，都是在redis内存操作的，那如何同步到数据库呢？  
如果不同步到数据库，就会出现数据丢失，请思考：如何把阅读量PV同步到mydql数据库?