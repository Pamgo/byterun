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

答：抛硬币出现正面的概率是<font  face="黑体" color="red">二分之一</font>,001就是要抛三次，每次的概率是二分之一，3次的概率就是三个二分之一相乘=八分之一。即八分之一的概率才会出现001，所以大概率8轮才能出现001。

在概率统计学里面，一行试验过程称之为<font face="黑体" color="red">伯努利试验</font>,它的原理就是`一直抛硬币，直到出现正面为止，整个过程称之为一次伯努利试验`。

抛了3次才出现正面，即001，记 k=3, 需要做概率n=8次 （8次伯努利试验）

抛了2次才出现正面，即01，记 k=2, 需要做概率n=4次 （4次伯努利试验）

抛了4次才出现正面，即0001，记 k=4, 需要做概率n=16次 （16次伯努利试验）

。。。

观察以上，我们可以很容易得出2个结论：

1. n次伯努利试验，出现k的最大值记为 k_max , 在 n 次试验中，抛次数不会大于 k_max

2. n次伯努利试验，至少又一次抛次数等于 k_max.

   

经过以上的结论，发现 n 和 k_max 中存在关联刮泥： <font face="黑体" color="red">n=2^k_max</font>

得出结论：只要知道k，就能算出n

那么 HyperLogLog 和 伯努利试验又什么关系呢？

HyperLogLog其实就是在伯努利试验的基础上做了改进。

### 3. HyperLogLog 的内存空间设计

![image](https://img-blog.csdnimg.cn/20200408211420492.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70)



### 4.剖析pfadd存储的设计原理

既然知道了HyperLogLog的内存空间设计原理，那接下来就可以分析如何把数据加入到这个空间

当执行`pfadd key value1` 时

<font face="黑体" color="red">步骤1:</font> 要把value1存储首先要计算到底存在<font color="red">16384</font>个桶的哪一个桶？

先把value1进行hash成6位，即64bit串（如下），前14位的二进制会被转换为10进制来作为桶号。

value1 = <font face="黑体" color="green">0000010000  0000010000  0000100000  0000010000 1000000000</font><font face="黑体" color="red"> 0000000000 0011</font>

------

这里有个问题：为什么是二进制前14位作为桶的编号呢？

因为HyperLogLog把内存空间分了<font color="red">16384</font>个桶,而 2^14=16384，这样14位的二进制转换为10进制最大值就是16384。

例如：字符串value 的前14位00 0000 0000 0011，其十进制为3，那么就会把字符串value存储到变化为3的桶内。

<font color="red">步骤2:</font> 既然已经计算出桶号了，那如何把数据填入桶？

在步骤1的时候，我们已经把value转换为hash 64位，并从前14位计算出了桶号，那剩下的50位，如何处理？

50位的存储方案：从右到左找到第一个1的坐标，例如value1从右到左找到第一个1是10，就把10转换为二进制001010，存入3号桶。

<font color="green">问题1:</font> 每个桶6个bit存满了怎么办？

这个问题不用担心，存不满的，即使最极端的情况下第50位才出现1，那就是50，转换为二进制110010，所以不用担心存满的问题。

<font color="green">问题2:</font> 相同一个桶，新加入数据如何存储？

例如；我们加入value2,前14位一样落到了3号桶里面，但是value2的后50位在第11位才找到1，11比原先的10大，故把11转换为二进制覆盖原来10的二进制。

如果value2的50位5才出现1，5比原先的10小，就不用处理。

所以采用了比较原则，谁大谁填进桶里。

```
总结：每个桶里面存的是最大值，即伯努利试验的k_max
```



### 5. 剖析pfcount统计的设计原理

![image](https://img-blog.csdnimg.cn/20200408214026107.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70)

### 6. HyperLogLog 更精准的概率优化

![image1](https://img-blog.csdnimg.cn/20200408214627538.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70)

![image2](https://img-blog.csdnimg.cn/2020040821464677.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70)

![image3](https://img-blog.csdnimg.cn/20200408214713917.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70)



### 7. 案例实战:基于Redis的UV计算

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

