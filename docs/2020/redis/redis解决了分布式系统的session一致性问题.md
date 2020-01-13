# redis解决了分布式系统的session一致性问题
### 一、Session有什么作用？
Session 是客户端与服务器通讯会话跟踪技术，服务器与客户端保持整个通讯的会话基本信息。  
客户端在第一次访问服务端的时候，服务端会响应一个sessionId并且将它存入到本地cookie中，
在之后的访问会将cookie中的sessionId放入到请求头中去访问服务器，  
如果通过这个sessionid没有找到对应的数据,那么服务器会创建一个新的sessionid并且响应给客户端。

### 二、分布式session有什么问题？

### 三、案例实战：Springboot实现用户登录session管理功能
#### 步骤1：加入springboot的常用依赖包
```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.8</version>
    </dependency>
```
#### 步骤2：编码实现用户登录逻辑
核心代码 
```java
session.setAttribute(session.getId(), user);
session.removeAttribute(session.getId());
```
```java
@Slf4j
@RestController
@RequestMapping(value = "/user")
public class UserController {

    Map<String, User> userMap = new HashMap<>();

    public UserController() {
        //初始化2个用户，用于模拟登录
        User u1=new User(1,"byterun1","byterun1");
        userMap.put("byterun1",u1);
        User u2=new User(2,"agan2","agan2");
        userMap.put("agan2",u2);
    }

    @GetMapping(value = "/login")
    public String login(String username, String password, HttpSession session) {
        //模拟数据库的查找
        User user = this.userMap.get(username);
        if (user != null) {
            if (!password.equals(user.getPassword())) {
                return "用户名或密码错误！！！";
            } else {
                session.setAttribute(session.getId(), user);
                log.info("登录成功{}",user);
            }
        } else {
            return "用户名或密码错误！！！";
        }
        return "登录成功！！！";
    }

    /**
     * 通过用户名查找用户
     */
    @GetMapping(value = "/find/{username}")
    public User find(@PathVariable String username) {
        User user=this.userMap.get(username);
        log.info("通过用户名={},查找出用户{}",username,user);
        return user;
    }

    /**
     *拿当前用户的session
     */
    @GetMapping(value = "/session")
    public String session(HttpSession session) {
        log.info("当前用户的session={}",session.getId());
        return session.getId();
    }

    /**
     * 退出登录
     */
    @GetMapping(value = "/logout")
    public String logout(HttpSession session) {
        log.info("退出登录session={}",session.getId());
        session.removeAttribute(session.getId());
        return "成功退出！！";
    }

}
```
#### 步骤3：编写session拦截器
session拦截器的作用：
验证当前用户发来的请求是否有携带sessionid，如果没有携带，提示用户重新登录。

#### 步骤4：体验测试
http://127.0.0.1:9090/user/login?username=byterun1&password=byterun1
http://127.0.0.1:9090/user/find/byterun1

### 四、采用docker安装部署Nginx
在主机192.168.1.138下，安装nginx，docker 的安装命令如下：
``` 
docker  run \
-d \
-p 8080:80 \
--name session-nginx \
nginx
```
- -d：在后台运行
- -p：容器的80端口映射到物理机的8080端口
- --name：容器的名字为session-nginx


- 关于为什么要用docker安装？<br>
因为用docker安装非常方便，docker安装1分钟不到就安装完，如果用传统安装，会各自安装包和命令，
最起码需要10分钟才能安装完。所以建议大家用docker安装。

- 不懂docker怎么办？<br>
关于不懂docker的同学，我已经准备了一份免费的docker教程，非常简单，大家去学一下就好。
https://study.163.com/course/courseMain.htm?share=2&shareId=1016671292&courseId=1209512823


体验:http://192.168.1.138:8080/


### 五、采用docker部署Nginx的集群负载均衡
#### 步骤1：在物理机建3个文件夹目录
- /data/volume/nginx/www    存放nginx的静态文件
- /data/volume/nginx/config 存放nginx的配置文件
- /data/volume/nginx/logs   存放nginx的日志

#### 步骤2：把nginx容器的配置文件拷贝出来
命令如下：
``` 
docker cp session-nginx:/etc/nginx/nginx.conf /data/volume/nginx/config/
docker cp session-nginx:/etc/nginx/conf.d/default.conf /data/volume/nginx/config/
```

#### 步骤3：修改nginx的集群负载均衡配置文件

为达到负载均衡目的，需要修改主机配置文件/data/volume/nginx/config/default.conf

- server外部追加，
```
upstream web {
      server 192.168.1.8:9090;
      server 192.168.1.8:9091; 
}
```
192.168.1.8代表Springboot用户登录服务部署主机

- server里边增加
``` 
location = / {
      proxy_pass http://web;
}
```
修改后 完整的default.conf源码，如下：
``` 
upstream web {
      server 192.168.1.8:9090;
      server 192.168.1.8:9091;
}

server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        proxy_pass http://web;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```


#### 步骤4：启动nginx
- 删除原先安装的nginx容器，不然会报错
docker rm -f session-nginx

- 启动nginx容器

```shell 
docker  run \
-d \
-p 8080:80 \
--name session-nginx \
-v /data/volume/nginx/www:/usr/share/nginx/html \
-v /data/volume/nginx/config/default.conf:/etc/nginx/conf.d/default.conf \
-v /data/volume/nginx/config/nginx.conf:/etc/nginx/nginx.conf \
-v /data/volume/nginx/logs:/var/log/nginx \
nginx
```
-v的意思就是把目标目录，映射到容器文件目录，例如：把容器的/var/log/nginx目录映射到主机的/data/volume/nginx/logs目录


### 六、剖析SpringBoot+Nginx的分布式Session不一致性
#### 步骤1：启动SpringBoot用户登录服务
把Springboot用户登录服务，启动2个服务，端口分别为9090和 9091

#### 步骤2：用IE体验效果

http://192.168.1.138:8080/user/login?username=byterun1&password=byterun1
http://192.168.1.138:8080/user/find/byterun1

结论：
1. 用户第一次访问Nginx，请求落到了服务器A，服务器A生成了一个sessionId,并保存在用户的cookie中。
2. 用户第二次再来访问Nginx，它这次把cookie里面的sessionId加入http的请求头中，这时请求落到了服务器B，服务器B发现没有找到sessionId,于是创建了一个新的sessionId并保存在用户的cookie中。
以上2个步骤，在分布式系统中，必将导致session错乱。


### 七、案例实战：SpringSession+redis解决分布式session不一致性问题
#### 步骤1：加入SpringSession、redis的依赖包
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
    <version>1.4.7.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```
#### 步骤2：修改配置文件
```properties

# 为某个包目录下 设置日志
logging.level.com.agan=debug


# 设置session的存储方式，采用redis存储
spring.session.store-type=redis

# session有效时长为10分钟
server.servlet.session.timeout=PT10M

## Redis 配置
## Redis数据库索引（默认为0）
spring.redis.database=0
## Redis服务器地址
spring.redis.host=192.168.1.138
## Redis服务器连接端口
spring.redis.port=6379
## Redis服务器连接密码（默认为空）
spring.redis.password=
```


### 八、剖析SpringSession的redis原理

#### 步骤1：分析SpringSession的redis数据结构
```shell 
127.0.0.1:6379> keys *
1) "spring:session:expirations:1578227700000"
2) "spring:session:sessions:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
3) "spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
```
共同点：3个key都是以spring:session:开头的，代表了SpringSession的redis数据。
"spring:session:sessions:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"

```shell
127.0.0.1:6379> type "spring:session:sessions:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
hash
```
127.0.0.1:6379> hgetall "spring:session:sessions:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"

//失效时间 100分钟
1) "maxInactiveInterval"
2) "\xac\xed\x00\x05sr\x00\x11java.lang.Integer\x12\xe2\xa0\xa4\xf7\x81\x878\x02\x00\x01I\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x17p"

// sesson的属性，存储了user对象
3) "sessionAttr:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
4) "\xac\xed\x00\x05sr\x00\x1ecom.agan.redis.controller.User\x16\"_m\x1b\xa0W\x7f\x02\x00\x03I\x00\x02idL\x00\bpasswordt\x00\x12Ljava/lang/String;L\x00\busernameq\x00~\x00\x01xp\x00\x00\x00\x01t\x00\x05byterun1q\x00~\x00\x03"

// session的创建时间
5) "creationTime"
6) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01ouW<K"

//最后的访问时间
7) "lastAccessedTime"
8) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01ouW<L"

#### 步骤2：分析SpringSession的redis过期策略
对于过期数据，一般有三种删除策略：
1. 定时删除，即在设置键的过期时间的同时，创建一个定时器， 当键的过期时间到来时，立即删除。
2. 惰性删除，即在访问键的时候，判断键是否过期，过期则删除，否则返回该键值。
3. 定期删除，即每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。
​redis 删除过期数据采用的是懒性删除+定期删除组合策略，也就是数据过期了并不会及时被删除。  
但由于redis是单线程，并且redis对删除过期的key优先级很低；如果有大量的过期key，就会出现key已经过期但是未删除。  

为了实现 session 过期的及时性，spring session 采用了定时删除+惰性删除的策略。
##### 定时删除
"spring:session:expirations:1578227700000"

```shell
127.0.0.1:6379> type "spring:session:expirations:1578228240000"
set
127.0.0.1:6379> smembers "spring:session:expirations:1578228240000"
1) "\xac\xed\x00\x05t\x00,expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
```
springsession 定时（1分钟）轮询，删除spring:session:expirations:[?] 的过期members

例如：spring:session:expirations:1578228240000 的1578228240000=2020-01-05 20:44:00:000 即在2020-01-05 20:44:00:000过期。
springsesion 定时检测超过2020-01-05 20:44:00:000 就删除spring:session:expirations:1578228240000的members的值
sessionId=5eddb9a3-5b1e-4bdd-a289-394b6d42388e
即删除
```shell 
1) "spring:session:expirations:1578228240000"
2) "spring:session:sessions:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
3) "spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
```
##### 惰性删除
spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e
```shell 
127.0.0.1:6379> type spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e
string
127.0.0.1:6379> get spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e
""
127.0.0.1:6379> ttl spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e
(integer) 4719
```

访问 

spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e的时候，判断key是否过期，过期则删除，否则返回改进的值。

例如 访问

spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e的时候

判断 ttl spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e是否过期，过期就直接删除 

```shell 
1) "spring:session:expirations:1578228240000"
2) "spring:session:sessions:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
3) "spring:session:sessions:expires:5eddb9a3-5b1e-4bdd-a289-394b6d42388e"
```






