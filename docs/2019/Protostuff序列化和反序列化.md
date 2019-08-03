## Protostuff序列化和反序列化
序列化和反序列化是在应对网络编程最常见的问题之一。
序列化就是将java对象转化成byte[];反序列化就是将byte[]转化成java对象，Protostuff属于一个更高效的序列化库。
## 引入依赖
引入依赖如下：

	        <dependency>
                <groupId>io.protostuff</groupId>
                <artifactId>protostuff-core</artifactId>
                <version>1.6.0</version>
            </dependency>
            <dependency>
                <groupId>io.protostuff</groupId>
                <artifactId>protostuff-runtime</artifactId>
                <version>1.6.0</version>
            </dependency>

![类方法](https://github.com/Pamgo/byterun/blob/master/docs/_media/ProstuffIOUtil.png)

## 使用方式

```/**
    * 构建RuntimeSchema,用于序列化、反序列化Seckill对象
    */
   public class MyRuntimeSchema {
       private static MyRuntimeSchema ourInstance = new MyRuntimeSchema();
   
       private RuntimeSchema<Seckill> goodsRuntimeSchema;
   
   
       public static MyRuntimeSchema getInstance() {
           return ourInstance;
       }
   
       private MyRuntimeSchema() {
           RuntimeSchema<Seckill> seckillSchemaVar = RuntimeSchema.createFrom(Seckill.class);
           setGoodsRuntimeSchema(seckillSchemaVar);
       }
   
       public RuntimeSchema<Seckill> getGoodsRuntimeSchema() {
           return goodsRuntimeSchema;
       }
   
       private void setGoodsRuntimeSchema(RuntimeSchema<Seckill> goodsRuntimeSchema) {
           this.goodsRuntimeSchema = goodsRuntimeSchema;
       }
   }
```

序列化对象：
```
对seckill进行序列化
   byte[] goodsBytes = ProtostuffIOUtil.toByteArray(seckill, MyRuntimeSchema.getInstance().getGoodsRuntimeSchema(),
                    LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE));
            jedis.set(seckillGoodsKey.getBytes(), goodsBytes);```
```
反序列化（redis）
```
@Repository
public class RedisDAO {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Resource(name = "initJedisPool")
    private JedisPool jedisPool;

    private RuntimeSchema<Seckill> schema = MyRuntimeSchema.getInstance().getGoodsRuntimeSchema();

    public Seckill getSeckill(long seckillId) {
        //redis操作逻辑
        try {
            Jedis jedis = jedisPool.getResource();
            try {
                String key = RedisKeyPrefix.SECKILL_GOODS + seckillId;
                byte[] bytes = jedis.get(key.getBytes());
                //缓存中获取到bytes
                if (bytes != null) {
                    //空对象
                    Seckill seckill = schema.newMessage();
                    // 反序列化为seckill
                    ProtostuffIOUtil.mergeFrom(bytes, seckill, schema);
                    //seckill 被反序列化
                    return seckill;
                }
            } finally {
                jedis.close();
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return null;
    }

```