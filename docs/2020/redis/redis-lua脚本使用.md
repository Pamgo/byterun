## 代码转换处理
```java

    @GetMapping(value = "/updateuser")
    public void updateUser(Integer uid,String uname) {
        String key="user:"+uid;
        //优化点：第一次发送redis请求
        String old=this.stringRedisTemplate.opsForValue().get(key);
        if(StringUtils.isEmpty(old)){
            //优化点：第二次发送redis请求
            this.stringRedisTemplate.opsForValue().set(key,uname);
            return;
        }
        if(old.equals(uname)){
            log.info("{}不用修改", key);
        }else{
            log.info("{}从{}修改为{}", key,old,uname);
            //优化点：第二次发送redis请求
            this.stringRedisTemplate.opsForValue().set(key,uname);
        }
        /*
        以上代码，看似简单，但是在高并发的情况下，还是有一点性能瓶颈，在性能方面主要是发送了2次redis请求。
        那如何优化呢？
        我们可以采用lua技术，把2次redis请求合成一次。
         */
    }
```
把以上代码优化成lua脚本
```lua

-- 成功设置返回1 没设置返回0
-- 如果redis没找到，就直接写进去
if redis.call('get', KEYS[1]) == nil then
   redis.call('set', KEYS[1], ARGV[1]);
   return 1
end
-- 如果旧值不等于新值，就把新值设置进去
if redis.call('get', KEYS[1]) ~= ARGV[1]  then
   redis.call('set', KEYS[1], ARGV[1]);
   return 1
else
   return 0
end
```
### 执行lua脚本
把以上代码保存为compareAndSet.lua user:101 , byterun101

```cmd
./redis-cli  --eval compareAndSet.lua user:101 , byterun101
```
1. --eval 告诉redis-cli 要执行后面的lua脚本，compareAndSet.lua脚本的目录位置
2. user:101 是redis要操作的key，在lua脚本中用KEYS[1]就能拿到
3. "," 逗号后面的byterun101 是lua的参数，lua脚本中用ARGV[1]就能拿到
注意： ","两边的空格不能省略，否则报错

执行效果如下：
```shell 
[root@node2 src]# ./redis-cli  --eval compareAndSet.lua user:101 , byterun101
(integer) 1  //第一次执行，redis没找到，就把值设置进去
[root@node2 src]# ./redis-cli  --eval compareAndSet.lua user:101 , byterun101
(integer) 0  //第二次执行，旧值和新值相同，返回0

```


### 案例实战：springboot实现多条redis命令合成一个lua
#### 步骤1：编写lua文件，并存储于resources/lua
把以下内容存储于resources/lua/compareAndSet.lua

```lua
-- 如果redis没找到，就把值写进去
if redis.call('get', KEYS[1]) == nil then
   redis.call('set', KEYS[1], ARGV[1]);
   return 1
end

-- 如果旧值不等于新增，就把新增设置进去
if redis.call('get', KEYS[1]) ~= ARGV[1]  then
   redis.call('set', KEYS[1], ARGV[1]);
   return 1
else
   return 0
end
```
#### 步骤2：创建lua脚本对象
创建DefaultRedisScript对象，用于存放lua脚本，把resources/lua/compareAndSet.lua脚本内容，存储在DefaultRedisScript对象里面。
```java 
@Configuration
public class LuaConfiguration {
  @Bean
    public DefaultRedisScript<Long> compareAndSetScript() {
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("lua/compareAndSet.lua")));
        redisScript.setResultType(Long.class);
        return redisScript;
    }
}
```

#### 步骤3：SpringBoot执行lua脚本
```java 
@GetMapping(value = "/updateuserlua")
public void updateUserLua(Integer uid,String uname) {
    String key="user:"+uid;
    //设置redis的key
    List<String> keys = Arrays.asList(key);
    //执行lua脚本，execute方法有3个参数，第一个参数是lua脚本对象，第二个是key列表，第三个是lua的参数数组
    Long n = this.stringRedisTemplate.execute(this.compareAndSetScript, keys, uname);
    if (n == 0) {
        log.info("{}不用修改", key);
    } else {
        log.info("{}修改为{}", key,uname);
    }
}
```

#### 步骤4：体验
http://127.0.0.1:9090/swagger-ui.html
```cmd
 [nio-9090-exec-8] c.a.r.c.CompareAndSetController          : user:100修改为abc
2019-12-28 20:10:34.396  INFO 8540 --- [nio-9090-exec-9] c.a.r.c.CompareAndSetController          : user:100不用修改
2019-12-28 20:10:34.546  INFO 8540 --- [io-9090-exec-10] c.a.r.c.CompareAndSetController          : user:100不用修改
2019-12-28 20:10:34.721  INFO 8540 --- [nio-9090-exec-1] c.a.r.c.CompareAndSetController          : user:100不用修改
2019-12-28 20:10:34.833  INFO 8540 --- [nio-9090-exec-2] c.a.r.c.CompareAndSetController          : user:100不用修改
2019-12-28 20:11:07.304  INFO 8540 --- [nio-9090-exec-4] c.a.r.c.CompareAndSetController          : user:100修改为agan 
```




