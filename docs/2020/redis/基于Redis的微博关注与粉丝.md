# 基于Redis的微博关注与粉丝

### 一、微博关注与粉丝的业务场景分析
阿甘关注了雷军：阿甘就是雷军的粉丝follower
雷军被阿甘关注：雷军就是阿甘的关注followee

### 二、微博关注与粉丝的redis技术方案
技术方案：每个用户都有2个集合：关注集合和粉丝集合
例如 阿甘关注了雷军，做了2个动作

1.把阿甘的userid加入雷军的粉丝follower集合set
2.把雷军的userid加入阿甘的关注followee集合set
 集合key设计
 阿甘的关注集合 key=followee:阿甘userid
 雷军的粉丝集合 key=follower：雷军userid
![微博关注与粉丝业务场景分析](https://img-blog.csdnimg.cn/20200316220851393.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70)

### SpringBoot+Redis 实现微博关注与粉丝
#### 功能1：关注

```java 
    /**
     * 阿甘关注了雷军，阿甘即使雷军的粉丝 follower
     * userId=阿甘
     * followeeId=雷军
     */
    @ApiOperation(value="关注")
    @PostMapping(value = "/follow")
    public void follow(Integer userId,Integer followeeId) {
        relationService.follow(userId,followeeId);
    }
```

```java
    /**
     * 阿甘关注了雷军
     */
    public void follow(Integer userId,Integer followeeId){
        SetOperations<String, Integer> opsForSet = redisTemplate.opsForSet();
        //阿甘的关注集合
        String followeekey = Constants.CACHE_KEY_FOLLOWEE + userId;
        //把雷军的followeeId，加入阿甘的关注集合中
        opsForSet.add(followeekey,followeeId);

        //雷军的粉丝集合
        String followerkey=Constants.CACHE_KEY_FOLLOWER+followeeId;
        //把阿甘的userid加入雷军的粉丝follower集合set
        opsForSet.add(followerkey,userId);

    }
```
#### 功能2：我的关注

```java 
    @ApiOperation(value="查看我的关注")
    @GetMapping(value = "/myFollowee")
    public List<UserVO> myFollowee(Integer userId){

        return this.relationService.myFollowee(userId);
    }
    
    /**
     * 查看我的关注
     */
    public List<UserVO> myFollowee(Integer userId){
        SetOperations<String, Integer> opsForSet = redisTemplate.opsForSet();
        //关注集合
        String followeekey = Constants.CACHE_KEY_FOLLOWEE + userId;
        Set<Integer> sets= opsForSet.members(followeekey);
        return this.getUserInfo(sets);
    }
```

#### 功能3：我的粉丝

```java 
    @ApiOperation(value="查看我的粉丝")
    @GetMapping(value = "/myFollower")
    public List<UserVO> myFollower(Integer userId){
        return this.relationService.myFollower(userId);
    }
```

```java 
    /**
     * 查看我的粉丝
     */
    public List<UserVO> myFollower(Integer userId){
        SetOperations<String, Integer> opsForSet = redisTemplate.opsForSet();
        //粉丝集合
        String followerkey=Constants.CACHE_KEY_FOLLOWER+userId;
        Set<Integer> sets= opsForSet.members(followerkey);

        return this.getUserInfo(sets);
    }
```
### 三、体验效果
用户和id的数据库对应关系

阿甘->id=1  
雷军->id=2  
王石->id=3  
潘石岂->id=4  

关注顺序

1. 阿甘关注：雷军 王石 潘石岂
1. 雷军关注：王石 潘石岂
1. 王石关注: 雷军
1. 潘石岂关注：王石