## stream消息队列实战
### 订单服务与积分服务的stream消息队列

### 为什么要用消息组？它解决了什么问题？
用过消息队列的同学应该知道目前比较流行的有RabbitMq、kafka等。
现在redis5.0 也完美支持消息队列。
例如 电商的订单，用户下完订单号，就要给用户添加积分 和发 app push消息.
如图1 以上这张图看上去，表面没有问题，但是部署集群的情况下就有问题了，如图2：



积分服务 和 push服务都部署集群的情况下，我们来演示会出什么问题：

1.订单服务发送3条订单数据
``` powershell
39.100.196.99:6379> multi
OK
(1.07s)
39.100.196.99:6379> xadd ordermq * orderno 10001
QUEUED
39.100.196.99:6379> xadd ordermq * orderno 10002
QUEUED
(1.94s)
39.100.196.99:6379> xadd ordermq * orderno 10003
QUEUED
39.100.196.99:6379> exec
1) "1587782682562-0"
2) "1587782682562-1"
3) "1587782682562-2"
```
2.集群中的第一个积分服务消费

``` powershell
39.100.196.99:6379> xread streams ordermq 0
1) 1) "ordermq"
   2) 1) 1) "1587782682562-0"
         2) 1) "orderno"
            2) "10001"
      2) 1) "1587782682562-1"
         2) 1) "orderno"
            2) "10002"
      3) 1) "1587782682562-2"
         2) 1) "orderno"
            2) "10003"
```
3.集群中的第二个积分服务消费
``` powershell
39.100.196.99:6379> xread streams ordermq 0
1) 1) "ordermq"
   2) 1) 1) "1587782682562-0"
         2) 1) "orderno"
            2) "10001"
      2) 1) "1587782682562-1"
         2) 1) "orderno"
            2) "10002"
      3) 1) "1587782682562-2"
         2) 1) "orderno"
            2) "10003"
```
观察以上数据，在集群的环境中，例如积分服务双实例，必然导致消息重复消费的问题。
为了解决这个问题，redis引入了消费组的概念。
什么是消息组呢？


### 案例实战：积分服务消息组
### 什么是消息组
如果你用过rabbitmq或 kafka就知道它的作用。
例如 积分服务在集群环境中部署了2台相同代码，都可以读取redis stream
如果直接采用xread读取，就会2台都会收到一模一样的数据。
为了解决，集群环境中，只有一台能消费消息,redis设计了消息组的概念。
消息组：一个消费组(group)内允许有多个消费者(consumer)（上面的直接执行 XREAD 指令的都是消费者），但是1条消息只会投递到其中一个 consumer上，
也就是组内每个 consumer 都会收到不同的消息。（这种模式术语叫做 集群模式）


#### 步骤1：先创建一个消息队列
沿用我们的消息队列，上节课的内容，队列名： ordermq
``` powershell
39.100.196.99:6379> xread streams ordermq 0
1) 1) "ordermq"
   2) 1) 1) "1587783541474-0"
         2) 1) "orderno"
            2) "10001"
      2) 1) "1587783541474-1"
         2) 1) "orderno"
            2) "10002"
      3) 1) "1587783541474-2"
         2) 1) "orderno"
            2) "10003"
```

#### 步骤2：创建一个消费组
XGROUP命令：
``` 
xgroup [CREATE key groupname id-or-$] [SETID key id-or-$] [DESTROY key groupname] [DELCONSUMER key groupname consumername]
```

CREATE创建组，SETID更新组起始消息ID，DESTROY销毁组，DELCONSUMER删除组内消费者等操作。
```
39.100.196.99:6379> xgroup CREATE ordermq ordergroup 0
OK
```
$意思从新消息开始分发，注意因为group是在服务端的结构，所以这里用的词是分发，而不是接收。
如果是填写id，代表消费，例如 填0，表示该组从第一条消息开始消费。

注意：
如果ordermq不存在时，会抛出以下异常，代表没要找到stream的key ordermq
 NOGROUP No such key 'ordermq' or consumer group 'ordergroup' in XREADGROUP with GROUP option
报这种错的解决方案有2种：
1. 先创建先创建一个消息队列  xadd ordermq * orderno 10001
2. 创建一个空流 ：XGROUP CREATE ordermq ordergroup $ MKSTREAM



#### 步骤3：多个消费者消费
XREADGROUP命令
``` 
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]
```

## 集群环境第一个积分服务进行消费
```powershell
39.100.196.99:6379> XREADGROUP GROUP ordergroup consumer-score  COUNT 2 STREAMS ordermq >
1) 1) "ordermq"
   2) 1) 1) "1587783541474-0"
         2) 1) "orderno"
            2) "10001"
      2) 1) "1587783541474-1"
         2) 1) "orderno"
            2) "10002"
```
## 集群环境第二个积分服务进行消费
``` powershell
39.100.196.99:6379> XREADGROUP GROUP ordergroup consumer-score  COUNT 2 STREAMS ordermq >
1) 1) "ordermq"
   2) 1) 1) "1587783541474-2"
         2) 1) "orderno"
            2) "10003"
```
语法说明：
`XREADGROUP group ordergroup consumer-score count 2 streams ordermq >`
消费者consumer-score,进入组ordergroup内，消费ordermq的数据
参数`>`表示未被组内消费的起始消息
参数`count 2`表示获取2条

组内消费的基本原理
STREAM类型会为每个组记录一个最后处理（交付）的消息ID（last_delivered_id），
这样在组内消费时，就可以从这个值后面开始读取，保证不重复消费。

练习题：
上面已经实现了积分服务的消费，剩下的是push服务的消费，参考积分服务的代码实现push服务的消费功能。

### 如何确保消息100%消费成功？

为了保证消息100%被消费成功（例如，组内消息读取但处理期间消费者崩溃导致的消息丢失）
STREAM 设计2个功能
1. Pending 列表，用于记录读取但并未处理完毕的消息。
2. XACK通知， 完成告知redis消息处理完成

### XPENDIING
命令XPENDIING 用来获消费组或消费内消费者的未处理完毕的消息。
```
XPENDING key group [start end count] [consumer]
```



``` powershell
39.100.196.99:6379> flushdb
OK
39.100.196.99:6379> multi
OK
39.100.196.99:6379> xadd ordermq * orderno 10001
QUEUED
39.100.196.99:6379> xadd ordermq * orderno 10002
QUEUED
39.100.196.99:6379> xadd ordermq * orderno 10003
QUEUED
39.100.196.99:6379> exec
1) "1588560150589-0"
2) "1588560150590-0"
3) "1588560150590-1"
(0.71s)
39.100.196.99:6379> xgroup CREATE ordermq ordergroup 0
OK

```
查看已读未处理的消息总数
```powershell
39.100.196.99:6379> XPENDING ordermq ordergroup
1) (integer) 0
2) (nil)
3) (nil)
4) (nil)
(1.77s)

39.100.196.99:6379> XREADGROUP GROUP ordergroup consumer-score  COUNT 2 STREAMS ordermq >
1) 1) "ordermq"
   2) 1) 1) "1588560150589-0"
         2) 1) "orderno"
            2) "10001"
      2) 1) "1588560150590-0"
         2) 1) "orderno"
            2) "10002"
(1.25s)
39.100.196.99:6379> XREADGROUP GROUP ordergroup consumer-score  COUNT 2 STREAMS ordermq >
1) 1) "ordermq"
   2) 1) 1) "1588560150590-1"
         2) 1) "orderno"
            2) "10003"

39.100.196.99:6379> XPENDING ordermq ordergroup
1) (integer) 3      #3个已读未处理的消息总数
2) "1588560150589-0" #开始id
3) "1588560150590-1" #结束id
4) 1) 1) "consumer-score" #消费者
      2) "3"              #consumer-scor消费者有3个未读消息
```
查看已读未处理的消息明细
```powershell
39.100.196.99:6379> XPENDING ordermq ordergroup - + 10
1) 1) "1588560150589-0"  #消息id
   2) "consumer-score"   #消费者
   3) (integer) 180924   #从读取到现在的时间
   4) (integer) 1        #消息被读取的次数，1次
2) 1) "1588560150590-0"
   2) "consumer-score"
   3) (integer) 180924
   4) (integer) 1
3) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 173746
   4) (integer) 1
(0.59s)
```

获取具体某个消费者的Pending列表
```powershell
39.100.196.99:6379> XPENDING ordermq ordergroup - + 10  consumer-score
1) 1) "1588560150589-0" #消息ID
   2) "consumer-score"  #所属消费者
   3) (integer) 305600  #IDLE，已读取时长
   4) (integer) 1       #delivery counter，消息被读取次数
2) 1) "1588560150590-0"
   2) "consumer-score"
   3) (integer) 305600
   4) (integer) 1
3) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 298422
   4) (integer) 1
```

上面的结果我们可以看到，我们之前读取的消息，都被记录在Pending列表中，说明全部读到的消息都没有处理，仅仅是读取了


### XACK
上面的结果我们可以看到，我们之前读取的消息，都被记录在Pending列表中，说明全部读到的消息都没有处理，仅仅是读取了。
那如何表示消费者处理完毕了消息呢？使用命令 XACK 完成告知消息处理完成，演示如下：

```powershell
39.100.196.99:6379> XPENDING ordermq ordergroup - + 10  consumer-score
1) 1) "1588560150589-0"
   2) "consumer-score"
   3) (integer) 450843
   4) (integer) 1
2) 1) "1588560150590-0"
   2) "consumer-score"
   3) (integer) 450843
   4) (integer) 1
3) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 443665
   4) (integer) 1
   
39.100.196.99:6379> xack ordermq ordergroup 1588560150589-0 #通知消息处理结束，用消息id表示
(integer) 1

39.100.196.99:6379> XPENDING ordermq ordergroup - + 10  consumer-score  #再次查看XPENDING列表
1) 1) "1588560150590-0"
   2) "consumer-score"
   3) (integer) 535677
   4) (integer) 1
2) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 528499
   4) (integer) 1
```

有了这样一个Pending机制，就意味着在某个消费者读取消息但未处理后，消息是不会丢失的。等待消费者再次上线后，
可以读取该Pending列表，就可以继续处理该消息了，保证消息的有序和不丢失。

练习题：
本节课老师实现了积分服务的消费，剩下的是push服务的消费，请同学们参考积分服务的代码 确保push服务消息100%消费成功。

### 没人消费的消息，采用消息转移
有这样一种场景：有些消息一直没消费处理，例如某个消费者宕机之后，没有办法再上线了，那么就需要将该消费者Pending的消息，
转移给其他的消费者处理，就是消息转移。
消息转移的操作时，将某个消息转移到其他的消费者的Pending列表中。
消息转移XCLAIM语法
```
xclaim key group consumer min-idle-time ID [ID ...] [IDLE ms] [TIME ms-unix-time] [RETRYCOUNT count] [force] [justid]
```
需要设置组、转移的目标消费者和消息ID，同时需要提供IDLE（已被读取时长），只有超过这个时长，才能被转移。

```powershell
# 查出当前 consumer-score未处理的消息
39.100.196.99:6379> xpending ordermq ordergroup - + 10 consumer-score
1) 1) "1588560150590-0"
   2) "consumer-score"
   3) (integer) 1238502 #超过了1238502ms未处理
   4) (integer) 1       #消息被读取了1次
2) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 1231324
   4) (integer) 1
   
# 转移超过60000ms的消息1588560150590-0 转移到consumerA的pending列表
39.100.196.99:6379> xclaim ordermq ordergroup consumerA 60000 1588560150590-0
1) 1) "1588560150590-0"  #转移成功的返回结果
   2) 1) "orderno"
      2) "10002"
(0.62s)

# 再次查看列表发现consumerA多了1条
39.100.196.99:6379> xpending ordermq ordergroup - + 10
1) 1) "1588560150590-0"
   2) "consumerA"        #consumerA已经把1588560150590-0转移到自己的队列中。
   3) (integer) 48236    #这个时间已经重置，不再是原来的时间
   4) (integer) 2        #消息被转移后，会累计1，即读取了2次。
2) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 1473655
   4) (integer) 1
```
练习题：
上面实现了积分服务的消费，剩下的是push服务的消费，参考积分服务的代码 实现push服务的消息转移。

### 删除死信消息
如果某个消息，不能被消费者处理，也就是不能被XACK，这是要长时间处于Pending列表中，即使被反复的转移给各个消费者也是如此。
此时该消息的delivery counter就会累加（上一节的例子可以看到），当累加到某个我们预设的临界值时，
我们就认为是坏消息（也叫死信，DeadLetter，无法投递的消息），
由于有了判定条件，我们将坏消息处理掉即可，删除即可。
删除一个消息，使用XDEL语法.
```powershell
#查看已读未处理的消息明细
39.100.196.99:6379> xpending ordermq ordergroup - + 10
1) 1) "1588560150590-0"
   2) "consumerA"
   3) (integer) 1256545
   4) (integer) 2
2) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 2681964
   4) (integer) 1
   
#查看队列 
39.100.196.99:6379> xrange ordermq - +
1) 1) "1588560150589-0"
   2) 1) "orderno"
      2) "10001"
2) 1) "1588560150590-0"
   2) 1) "orderno"
      2) "10002"
3) 1) "1588560150590-1"
   2) 1) "orderno"
      2) "10003"

# 使用xdel删除队列的消息
39.100.196.99:6379> xdel ordermq 1588560150590-0
(integer) 1
(0.87s)


39.100.196.99:6379> xrange ordermq - +
1) 1) "1588560150589-0"
   2) 1) "orderno"
      2) "10001"
2) 1) "1588560150590-1"
   2) 1) "orderno"
      2) "10003"

# 删除了队列后，居然未处理的消息居然还存在  
39.100.196.99:6379> xpending ordermq ordergroup - + 10
1) 1) "1588560150590-0"
   2) "consumerA"
   3) (integer) 1436364
   4) (integer) 2
2) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 2861783
   4) (integer) 1
#为了把未处理的消息列表，也删除，我们给他一个ack  
39.100.196.99:6379> xack ordermq ordergroup 1588560150590-0
(integer) 1
#最终删除成功了
39.100.196.99:6379> xpending ordermq ordergroup - + 10
1) 1) "1588560150590-1"
   2) "consumer-score"
   3) (integer) 3000787
   4) (integer) 1
```
练习题：
上面实现了积分服务的消费，剩下的是push服务的消费，参考积分服务的代码 实现push服务的死信消息。

## 6.案例实战：积分服务的消费队列 
### 步骤1：集群积分服务A消费
``` java

@Service
@Slf4j
public class ScoreConsumerA {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 消费任务
     */
    @PostConstruct
    public void consume() {
        log.info("启动消费 ..........");
        new Thread(() -> this.consumeData()).start();
    }


    public void consumeData() {


        StreamOffset streamOffset = StreamOffset.create(Constants.MQ_ORDER, ReadOffset.lastConsumed());
        //创建一个消费者
        Consumer consumer = Consumer.from(Constants.GROUP_SCORE, Constants.CONSUMER_SCORE);
        //block 阻塞 60s读
        StreamReadOptions streamReadOptions = StreamReadOptions.empty().block(Duration.ofMinutes(60));
        while (true) {
            //XREADGROUP GROUP group-score consumer-score  block 60000 STREAMS mq-order >
            List<MapRecord<String, String, String>> list = this.redisTemplate.opsForStream().read(consumer, streamReadOptions, streamOffset);
            for (MapRecord<String, String, String> obj : list) {
                log.info("A积分服务消费了{}", list);
                //TODO 添加积分逻辑

                //通知消息处理结束，用消息ID标识
                //xack ordermq ordergroup 1588560150589-0
                this.redisTemplate.opsForStream().acknowledge(Constants.MQ_ORDER, Constants.GROUP_SCORE, obj.getId());
            }

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}

```

### 步骤2：订单服务发消息
``` java
    /**
     * 发送消息
     */
    @GetMapping(value = "/add")
    public RecordId add(String orderno) {
        Map<String,String> map=new HashMap<>();
        map.put("orderno",orderno);
        RecordId recordId =this.redisTemplate.opsForStream().add(Constants.MQ_ORDER,map);
        return recordId;
    }
```



## 7.案例实战：积分服务+push服务的集群消费队列

### 步骤3：集群积分服务B消费

```java

@Service
@Slf4j
public class ScoreConsumerB {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 消费任务
     */
    @PostConstruct
    public void consume() {
        log.info("启动消费 ..........");
        new Thread(() -> this.consumeData()).start();
    }


    public void consumeData() {

        /*
         * 由于redisTemplate不支持检查group是否存在，同时也不支持创建XGROUP CREATE mq-order group-score $ MKSTREAM
         * 故；
         * 我们在启动程序之前，要在redis执行以下命令，先创建消息组，不然程序找不到消息组
         * XGROUP CREATE mq-order group-score $ MKSTREAM
         * 切记 不然会报错  NOGROUP No such key 'mq-order' or consumer group 'group-score' in XREADGROUP with GROUP option
         */
        StreamOffset streamOffset = StreamOffset.create(Constants.MQ_ORDER, ReadOffset.lastConsumed());
        //创建一个消费者
        Consumer consumer = Consumer.from(Constants.GROUP_SCORE, Constants.CONSUMER_SCORE);
        //block 阻塞 60s读
        StreamReadOptions streamReadOptions = StreamReadOptions.empty().block(Duration.ofMinutes(60));
        while (true) {
            //XREADGROUP GROUP group-score consumer-score  block 60000 STREAMS mq-order >
            List<MapRecord<String, String, String>> list = this.redisTemplate.opsForStream().read(consumer, streamReadOptions, streamOffset);
            for (MapRecord<String, String, String> obj : list) {
                log.info("B积分服务消费了{}", list);
                //TODO 添加积分逻辑

                //通知消息处理结束，用消息ID标识
                //xack ordermq ordergroup 1588560150589-0
                this.redisTemplate.opsForStream().acknowledge(Constants.MQ_ORDER, Constants.GROUP_SCORE, obj.getId());
            }

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}

```




### 步骤4：PUSH服务消费

```java

@Service
@Slf4j
public class PushConsumerA {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 消费任务
     */
    @PostConstruct
    public void consume() {
        log.info("启动消费 ..........");
        new Thread(() -> this.consumeData()).start();
    }


    public void consumeData() {

        /*
         * 由于redisTemplate不支持检查group是否存在，同时也不支持创建XGROUP CREATE mq-order group-score $ MKSTREAM
         * 故；
         * 我们在启动程序之前，要在redis执行以下命令，先创建消息组，不然程序找不到消息组
         * XGROUP CREATE mq-order group-push $ MKSTREAM
         * 切记 不然会报错  NOGROUP No such key 'mq-order' or consumer group 'group-score' in XREADGROUP with GROUP option
         */
        StreamOffset streamOffset = StreamOffset.create(Constants.MQ_ORDER, ReadOffset.lastConsumed());
        //创建一个消费者
        Consumer consumer = Consumer.from(Constants.GROUP_PUSH, Constants.CONSUMER_PUSH);
        //block 阻塞 60s读
        StreamReadOptions streamReadOptions = StreamReadOptions.empty().block(Duration.ofMinutes(60));
        while (true) {
            //XREADGROUP GROUP group-score consumer-score  block 60000 STREAMS mq-order >
            List<MapRecord<String, String, String>> list = this.redisTemplate.opsForStream().read(consumer, streamReadOptions, streamOffset);
            for (MapRecord<String, String, String> obj : list) {
                log.info("A-push服务消费了{}", list);
                //TODO 添加积分逻辑

                //通知消息处理结束，用消息ID标识
                //xack ordermq ordergroup 1588560150589-0
                this.redisTemplate.opsForStream().acknowledge(Constants.MQ_ORDER, Constants.GROUP_PUSH, obj.getId());
            }

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}

```

