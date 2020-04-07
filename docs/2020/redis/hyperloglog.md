### HyperLogLog 命令详解

HyperLogLog 目前只支持3个命令，PFADD、PFCOUNT、PFMERGE

#### PFADD

将元素加入到HyperLogLog数据结构中，如果 HyperLogLog 的基数估算值在命令执行之后出现了变化，那么命令返回1，否则返回0。

#### PFCOUNT

返回给定 HyperLogLog 的基数估算值。

```powershell
127.0.0.1:6379> pfadd uv 192.168.1.100 192.168.1.101 192.168.1.102 192.168.1.103
(integer) 1
127.0.0.1:6379> pfadd uv 192.168.1.100
(integer) 0
127.0.0.1:6379> pfcount uv
(integer) 4
```

#### PFMERGE

把多个HyperLogLog合并为一个HyperLogLog，合并后的HyperLogLog的基数是通过所有的HyperLogLog进行并集后，得出来的。

```powershell
127.0.0.1:6379> pfadd uv1 192.168.1.100 192.168.1.101 192.168.1.102 192.168.1.103
(integer) 1
127.0.0.1:6379> pfcount uv1
(integer) 4
127.0.0.1:6379> pfadd uv2 192.168.1.100 192.168.2.101 192.168.2.102 192.168.2.103
(integer) 1
127.0.0.1:6379> pfadd uv3 192.168.1.100 192.168.3.101 192.168.3.102 192.168.3.103
(integer) 1
127.0.0.1:6379> pfmerge aganuv uv1 uv2 uv3
OK
127.0.0.1:6379> pfcount aganuv
(integer) 10
```

这个命令很重要，例如你可以统计一周 或 一个月的uv 就可以使用此命令来轻易实现。

### 1. 淘宝首页亿级UV的技术方案分析

先打开淘宝首页，这个淘宝首页每天的UV大约又1.5亿，也就是说每天大约又1.5亿的ip去访问淘宝。

* 那么思考一个问题：针对这种一线互联网系统，如何统计一个日均1.5亿网站的UV量？

1. 首先要存储1.5亿个ip

2. 随便某个IP来访问你的网站，都要去查询当前这个IP是否已经存在1.5亿里面，不存在则可以加入。

   如果让你来设计，你会如何设计呢？下面将会分析不同的方案做法：

> 方案一：采用mysql来实现，建一张表来存储

缺陷：查询太慢，随着数据的累加，查询会越来越慢，而且存储的数据量也非常大。

> 方案二：采用redis的hash存储

`hash 存储的格式 = <key_day,<key_ip,1>>`

key_ip 是字符串存储每个IPv4 的地址最多是15个字节（IP 格式 = ‘xxx.xxx.xxx.xxx'，例如192.168.101.100)

1.5亿 * 15 个字节=2G， 一天的数据量就达到了2个G， 一个月就是60G， 这个方案对于redis来说，明显是不行的。另外如果要存储IPv6地址的话，那还需要更大的内存空间才行。

> 方案三：采用redis的HyperLogLog存储

redis专门设计了一种HyperLogLog来做<font face="黑体" color=red>基数</font>统计,HyperLogLog 是一个基数 <font face="黑体" color=red>估计</font>算法，只需要极少空间就可以计算超大基数；每个 HyperLogLog key占用的空间非常小，只需要12KB的内存，就可以统计出2^64个不同元素的基数。

12KB 就能统计出2^64个不同元素的基数，对于日均1.5亿简直就是小意思了。

但是这里有个小问题，要说清楚：HyperLogLog 是又误差的，标准误差是0.81%，也就是说 在0.81%的误差下能够统计2^64个数据。所以HyperLogLog很适合在比如日活量月活量、UV等此类对<font face="黑体" color="red">精准度不高</font>的场景。

它的设计原理就是：通过牺牲准确率来减少内存空间的消耗。

> 那什么是基数?

例如：集合｛a, a, b, b, c, d｝，这个集合的基数集为 {a, b, c, d}，基数就是不重复为4.

### 2. 伯努利实验

回顾一个高中概率论数学题：一个硬币又正反两面，一次实验出现2个反面一个正面，我们用0代表反面，1代表正面， 2个反面1个正面=001，那么大概率要进行多少次实验才能出现001呢？

TODO 

### 3.案例实战:基于Redis的UV计算

#### 步骤1：模拟UV访问

```java
@Service
@Slf4j
public class TaskService {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     *模拟UV访问
     */
    @PostConstruct
    public void init(){
        log.info("模拟UV访问 ..........");
        new Thread(()->this.refreshData()).start();
    }

    /**
     * 刷新当天的统计数据
     */
    public void refreshDay(){
        Random rand = new Random();
        String ip=rand.nextInt(999)+"."+
                rand.nextInt(999)+"."+
                rand.nextInt(999)+"."+
                rand.nextInt(999);

        //redis 命令 pfadd
        long n=this.redisTemplate.opsForHyperLogLog().add(Constants.PV_KEY,ip);
        log.debug("ip={} , returen={}",ip,n);
    }

    /**
     * 按天模拟UV统计
     */
    public void refreshData(){
        while (true){
            this.refreshDay();
            //TODO 在分布式系统中，建议用xxljob来实现定时
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}
```

#### 步骤2：实现UV统计功能

```java
@RestController
@Slf4j
public class Controller {

    @Autowired
    private RedisTemplate redisTemplate;
    

    @GetMapping(value = "/uv")
    public long uv() {
        //redis命令 pfcount
        long size=this.redisTemplate.opsForHyperLogLog().size(Constants.PV_KEY);
        return size;
    }

}
```

