### Mac系统Redis安装布隆过滤器

下载布隆过滤器：[地址](https://github.com/RedisBloom/RedisBloom)

> 1、解压RedisBloom压缩包放到系统对应的目录中，如图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425204519846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70#pic_center)

在当前目录执行命令

```shell
make
```

得到一个`redisbloom.so`文件。

> 2、打开mac中的redis安装目录得redis.conf文件，添加配置

```properties
loadmodule /usr/local/RedisBloom-master/redisbloom.so
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425204551127.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70#pic_center)

保存后，退出。

启动redis

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425204610634.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70#pic_center)


> 3、验证redis是否成功安装了布隆过滤器模块

执行以下命令后添加成功则代表安装模块成功

```shell
>redis-cli
>bf.add testkey "a"
1
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425204634252.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70#pic_center)