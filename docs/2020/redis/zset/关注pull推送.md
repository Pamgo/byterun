# 基于redis的微博个人首页
### 微博个人首页业务场景分析

### 微博个人首页的redis技术方案

### 3.案例实战:基于push技术，实现微博个人列表
#### 步骤1：发微博后，写入个人主页-队列
``` java
    /**
     * push到个人主页
     */
    public void pushHomeList(Integer userId,Integer postId){
        String key= Constants.CACHE_MY_POST_BOX_LIST_KEY+userId;
        this.redisTemplate.opsForList().leftPush(key,postId);
    }
```

#### 步骤2：查看个人列表
``` java
    /**
     * 获取个人主页列表
     */
    public PageResult<Content> homeList(Integer userId,int page, int size){
        PageResult<Content> pageResult=new PageResult();

        List<Integer> list=null;
        long start = (page - 1) * size;
        long end = start + size - 1;
        try {
            String key= Constants.CACHE_MY_POST_BOX_LIST_KEY+userId;
            //1.查询用户的总数
            int total=this.redisTemplate.opsForList().size(key).intValue();
            pageResult.setTotal(total);

            //2.采用redis list数据结构的lrange命令实现分页查询。
            list = this.redisTemplate.opsForList().range(key, start, end);

            //3.去拿明细
            List<Content> contents=this.getContents(list);
            pageResult.setRows(contents);
        }catch (Exception e){
            log.error("异常",e);
        }
        return pageResult;
    }
```

``` java

    protected List<Content> getContents(List<Integer> list){
        List<Content> contents=new ArrayList<>();

        //发布内容的key
        List<String> hashKeys=new ArrayList<>();
        hashKeys.add("id");
        hashKeys.add("content");
        hashKeys.add("userId");

        HashOperations<String, String ,Object> opsForHash=redisTemplate.opsForHash();
        for (Integer id:list){
            String hkey= Constants.CACHE_CONTENT_KEY+id;
            List<Object> clist=opsForHash.multiGet(hkey,hashKeys);
            //redis没有去db找
            if (clist.get(0)==null && clist.get(1)==null){
                Content obj=this.contentMapper.selectByPrimaryKey(id);
                contents.add(obj);
            }else{
                Content content=new Content();
                content.setId(clist.get(0)==null?0:Integer.valueOf(clist.get(0).toString()));
                content.setContent(clist.get(1)==null?"":clist.get(1).toString());
                content.setUserId(clist.get(2)==null?0:Integer.valueOf(clist.get(2).toString()));
                contents.add(content);
            }
        }
        return contents;
    }
```

#### 步骤3：体验
用户和id的数据库对应关系
阿甘->id=1  
雷军->id=2  
王石->id=3  
潘石岂->id=4  

关注顺序
阿甘关注：雷军 王石 潘石岂
雷军关注：王石 潘石岂
王石关注: 雷军
潘石岂关注：王石

雷军发微博：
第一条微博：123
第二条微博: abc
第一条微博：wwww

体验步骤：
1.雷军先发微博
2.查看雷军的个人主页




### 4.案例实战:基于push技术，实现微博关注列表
#### 步骤1：发一条微博，批量推送给所有粉丝
``` java
    /**
     * 发一条微博，批量推送给所有粉丝
     */
    private void pushFollower(int userId,int postId){
        SetOperations<String, Integer> opsForSet = redisTemplate.opsForSet();

        //读取粉丝集合
        String followerkey=Constants.CACHE_KEY_FOLLOWER+userId;
        //千万不能取set集合的所有数据，如果数据量大的话，会卡死
        // Set<Integer> sets= opsForSet.members(followerkey);
        Cursor<Integer> cursor = opsForSet.scan(followerkey, ScanOptions.NONE);
        try{
            while (cursor.hasNext()){
                //拿出粉丝的userid
                Integer object = cursor.next();
                String key= Constants.CACHE_MY_ATTENTION_BOX_LIST_KEY+object;
                this.redisTemplate.opsForList().leftPush(key,postId);

            }
        }catch (Exception ex){
            log.error("",ex);
        }finally {
            try {
                cursor.close();
            } catch (IOException e) {
                log.error("",e);
            }
        }
    }
```

#### 步骤2：查看关注列表
``` java
    /**
     * 获取关注列表
     */
    public PageResult<Content> attentionList(Integer userId,int page, int size){
        PageResult<Content> pageResult=new PageResult();

        List<Integer> list=null;
        long start = (page - 1) * size;
        long end = start + size - 1;
        try {
            String key= Constants.CACHE_MY_ATTENTION_BOX_LIST_KEY+userId;
            //1.设置总数
            int total=this.redisTemplate.opsForList().size(key).intValue();
            pageResult.setTotal(total);

            //2.采用redis,list数据结构的lrange命令实现分页查询。
            list = this.redisTemplate.opsForList().range(key, start, end);

            //3.去拿明细数据
            List<Content> contents=this.getContents(list);
            pageResult.setRows(contents);
        }catch (Exception e){
            log.error("异常",e);
        }
        return pageResult;
    }
```
#### 步骤3：体验
用户和id的数据库对应关系
阿甘->id=1  
雷军->id=2  
王石->id=3  
潘石岂->id=4  

关注顺序
阿甘关注：雷军 王石 潘石岂
雷军关注：王石 潘石岂
王石关注: 雷军
潘石岂关注：王石

雷军发微博：
第一条微博：今天出太阳了
第二条微博: 下雨了
第一条微博：出现彩虹了

体验步骤：
1.雷军先发微博
2.查看雷军的个人主页
3.阿甘查看推荐首页

###微博个人列表和关注列表的性能瓶颈

#### 微博个人和关注列表的性能瓶颈分析
上面已经实现了2个list，一个是个人list,一个是关注list
这2个list存在性能的问题，就是这个list无线增长。
大家想想，每个人有2个list,而这2个list每次发微博都会push到个人和关注list，
时间久了必定把redis撑爆对不对，那怎么办呢 ？限定list的长度
给大家看看微博和百度是怎么做到的？
微博限定20页
百度搜索限定了76页



#### 微博个人和关注列表的性能优化方案


优化方案采用：限定个人和关注list的长度为1000，即，
发微博的时候，往个人和关注list push完成后，把list的长度剪切为1000，
具体的技术方案采用list 的ltrim命令来实现。
LTRIM key start end
截取队列指定区间的元素,其余元素都删除






#### 案例实战:微博个人和关注列表的性能优化
``` java
//性能优化，截取前1000条
if(this.redisTemplate.opsForList().size(key)>1000){
    this.redisTemplate.opsForList().trim(key,0,1000);
}
```

### 3.案例实战:基于pull技术，实现微博个人列表
PullContentController
``` java

@Api(description = "微博内容：pull功能")
@RestController
@RequestMapping("/pull-content")
public class PullContentController {

    @Autowired
    private PullContentService contentService;

    @ApiOperation(value="用户发微博")
    @PostMapping(value = "/post")
    public void post(@RequestBody ContentVO contentVO) {
        Content content=new Content();
        BeanUtils.copyProperties(contentVO,content);
        contentService.post(content);
    }

    @ApiOperation(value="获取个人列表")
    @GetMapping(value = "/homeList")
    public PageResult<Content> getHomeList(Integer userId, int page, int size){
        return  this.contentService.homeList(userId,page,size);
    }

}

```
PullContentService
``` java
@Slf4j
@Service
public class PullContentService extends ContentService{

    /**
     * 用户发微博
     */
    public void post(Content obj){
        Content temp=this.addContent(obj);

        this.addMyPostBox(temp);
    }


    /**
     * 发布微博的时候，加入到我的个人列表
     */
    public void addMyPostBox(Content obj){
        String key= Constants.CACHE_MY_POST_BOX_ZSET_KEY+obj.getUserId();
        //按秒为单位
        long score=obj.getCreateTime().getTime()/1000;
        this.redisTemplate.opsForZSet().add(key,obj.getId(),score);
    }

    /**
     * 获取个人列表
     */
    public PageResult<Content> homeList(Integer userId, int page, int size){
        PageResult<Content> pageResult=new PageResult();
        List<Integer> list=new ArrayList<>();
        long start = (page - 1) * size;
        long end = start + size - 1;
        try {
            String key= Constants.CACHE_MY_POST_BOX_ZSET_KEY+userId;
            //1.设置总数
            long total=this.redisTemplate.opsForZSet().zCard(key);
            pageResult.setTotal(total);

            //2.分页查询
            //redis ZREVRANGE
            Set<ZSetOperations.TypedTuple<Integer>> rang= this.redisTemplate.opsForZSet().reverseRangeWithScores(key,start,end);
            for (ZSetOperations.TypedTuple<Integer> obj:rang){
                list.add(obj.getValue());
                log.info("个人post集合value={},score={}",obj.getValue(),obj.getScore());
            }

            //3.去拿明细数据
            List<Content> contents=this.getContents(list);
            pageResult.setRows(contents);
        }catch (Exception e){
            log.error("异常",e);
        }
        return pageResult;
    }

}

```

### 4.案例实战:基于pull技术，实现微博关注列表
``` java

    /**
     * 刷新拉取用户关注列表
     * 用户第一次刷新或定时刷新 触发
     */
    private void refreshAttentionBox(int userId){
        //获取刷新的时间
        String refreshkey=Constants.CACHE_REFRESH_TIME_KEY+userId;
        Long ago=(Long) this.redisTemplate.opsForValue().get(refreshkey);
        //如果时间为空，取2天前的时间
        if (ago==null){
            //当前时间
            long now=System.currentTimeMillis()/1000;
            //当前时间减去2天
            ago=now-60*60*24*2;
        }

        //提取该用户的关注列表
        String followerkey=Constants.CACHE_KEY_FOLLOWEE+userId;
        Set<Integer> sets= redisTemplate.opsForSet().members(followerkey);
        log.debug("用户={}的关注列表={}",followerkey,sets);

        //当前时间
        long now=System.currentTimeMillis()/1000;
        String attentionkey= Constants.CACHE_MY_ATTENTION_BOX_ZSET_KEY+userId;
        for (Integer id:sets){
            //去关注人的个人主页，拿最新微博
            String key= Constants.CACHE_MY_POST_BOX_ZSET_KEY+id;
            Set<ZSetOperations.TypedTuple<Integer>> rang= this.redisTemplate.opsForZSet().rangeByScoreWithScores(key,ago,now);
            if(!CollectionUtils.isEmpty(rang)){
                //加入我的关注post集合 就是通过上次刷新时间计算出最新的微博，写入关注zset集合；再更新刷新时间
                this.redisTemplate.opsForZSet().add(attentionkey,rang);
            }

        }

        //关注post集合 只留1000个
        //计算post集合，总数
        long count=this.redisTemplate.opsForZSet().zCard(attentionkey);
        //如果大于1000，就剔除多余的post
        if(count>1000){
            long end=count-1000;
            //redis ZREMRANGEBYRANK
            this.redisTemplate.opsForZSet().removeRange(attentionkey,0,end);
        }
        long days=this.redisTemplate.getExpire(attentionkey,TimeUnit.DAYS);
        if(days<10){
            //设置30天过期
            this.redisTemplate.expire(attentionkey,30,TimeUnit.DAYS);
        }

    }

```





### 体验


用户和id的数据库对应关系
阿甘->id=1  
雷军->id=2  
王石->id=3  
潘石岂->id=4  

关注顺序
阿甘关注：雷军 王石 潘石岂
雷军关注：王石 潘石岂
王石关注: 雷军
潘石岂关注：王石

1.雷军发微博：
第一条微博：小米手机涨价了100
第二条微博：小米手机降价1000
第三条微博：小米手机免费送啦  哈哈哈
2.体验个人列表，用雷军账户查看

3.体验关注列表，用阿甘账户查看