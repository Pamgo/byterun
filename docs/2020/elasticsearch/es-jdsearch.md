### springboot集成Elasticsearch7.x

项目地址：[SpringBoot+Elasticesearch7.x](https://github.com/Pamgo/sb-elasticsearch-demo.git)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.byterun</groupId>
    <artifactId>sb-elasticsearch-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>sb-elasticsearch-demo</name>
    <description>Demo project for Spring Boot</description>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.6.RELEASE</version>
        <relativePath />
    </parent>
    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <elasticsearch.version>7.5.1</elasticsearch.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.12</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.68</version>
        </dependency>
        <!--视频音乐爬取tika-->
        <!--解析网页-->
        <dependency>
            <groupId>org.jsoup</groupId>
            <artifactId>jsoup</artifactId>
            <version>1.13.1</version>
        </dependency>
    </dependencies>
<!--
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>-->

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>2.2.6.RELEASE</version>
            </plugin>
        </plugins>
    </build>

</project>

```

#### es 配置类

```java
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestHighLevelClient;
@Configuration
public class ElasticSearchClientConfig {

    @Bean
    @Qualifier("highLevelClient")
    public RestHighLevelClient restHighLevelClient() {
        RestHighLevelClient highLevelClient = new RestHighLevelClient(RestClient.builder(
                new HttpHost("127.0.0.1", 9200, "http")
        ));
        return highLevelClient;
    }
}

```



#### Utils工具

```java

public class HtmlParseUtil {

//    public static void main(String[] args) throws IOException {
//        List<ContentModel> list = parseHtml("java");
//        System.out.println(JSON.toJSONString(list));
//
//    }

    public static List<ContentModel> parseHtml(String keyword) throws IOException {
        // 获取请求
        String url = "https://search.jd.com/Search?keyword="+keyword;
        // 解析网页,jsoup返回document对象
        Document document = Jsoup.parse(new URL(url), 30000);
        // 所有js中使用的方法
        Element j_goodsList = document.getElementById("J_goodsList");
        //System.out.println(j_goodsList.html());
        // 获取所有的li元素
        Elements li = j_goodsList.getElementsByTag("li");
        List<ContentModel> list = new ArrayList<>();
        for (Element element : li) {
            // 图片延迟加载（无法获取），可以获取懒加载地址
            //String img = element.getElementsByTag("img").eq(0).attr("src");
            String img = element.getElementsByTag("img").eq(0).attr("source-data-lazy-img");
            String price = element.getElementsByClass("p-price").eq(0).text();
            String title = element.getElementsByClass("p-name").eq(0).text();
            ContentModel model = new ContentModel();
            model.setImg(img)
                    .setPrice(price)
                    .setTitle(title);
            list.add(model);
        }

        return list;
    }
}
```

#### service层

```java
import lombok.extern.slf4j.Slf4j;
import org.elasticsearch.action.bulk.BulkRequest;
import org.elasticsearch.action.bulk.BulkResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.text.Text;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.common.xcontent.XContentType;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.index.query.TermQueryBuilder;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.fetch.subphase.highlight.HighlightBuilder;
import org.elasticsearch.search.fetch.subphase.highlight.HighlightField;

@Service
@Slf4j
public class ContentService {
    @Autowired
    private RestHighLevelClient restHighLevelClient;

    // 1、解析数据放入es中
    public boolean parseContent(String keywords) throws IOException {
        List<ContentModel> parseHtml = HtmlParseUtil.parseHtml(keywords);
        // 把查询的数据放入 es 中
        BulkRequest bulkRequest = new BulkRequest();
        bulkRequest.timeout(TimeValue.timeValueSeconds(2));

        parseHtml.forEach((c)-> {
            bulkRequest.add(new IndexRequest(EsConst.indices_jd)
            .source(JSON.toJSONString(c), XContentType.JSON));
        });

        BulkResponse bulkResponse = restHighLevelClient.bulk(bulkRequest, RequestOptions.DEFAULT);
        return !bulkResponse.hasFailures();
    }

    // 分页查询
    public List<Map<String, Object>> serachPage(String keyword, int pageNo,
                                                int pageSize) throws IOException {
        if (pageNo <= 1) {
            pageNo = 1;
        }

        // 条件查询
        SearchRequest searchRequest = new SearchRequest(EsConst.indices_jd);
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        // 分页
        sourceBuilder.from(pageNo);
        sourceBuilder.size(pageSize);

        //精准匹配
        TermQueryBuilder termQuery = QueryBuilders.termQuery("title", keyword);
        sourceBuilder.query(termQuery);
        sourceBuilder.timeout(TimeValue.timeValueSeconds(50));
        // 高亮
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.field("title");
        highlightBuilder.requireFieldMatch(false); // 是否需要多个高亮
        highlightBuilder.preTags("<span type='color:red");
        highlightBuilder.postTags("</span>");
        sourceBuilder.highlighter(highlightBuilder);
        // 执行搜索，分页
        searchRequest.source(sourceBuilder);

        SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析结果，
        List<Map<String, Object>> list = new ArrayList<>();
        for (SearchHit hit : searchResponse.getHits().getHits()){
            // 解析高亮字段
            Map<String, HighlightField> highlightFields =
                    hit.getHighlightFields();
            HighlightField title = highlightFields.get("title");
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            if (title != null) { // 解析高亮,替换高亮字段
                Text[] fragments = title.fragments();
                String n_title = "";
                for (Text t : fragments) {
                    n_title += t;
                }
                sourceAsMap.put("title", n_title); // 替换原理的高亮字段
            }
            list.add(sourceAsMap);
        }

        return list;
    }

}

```

#### controller层

```java
 // 京东查询、爬虫,数据入库
    @GetMapping("/parseJdSearch/{keyword}")
    public boolean parseJdSearch(@PathVariable("keyword") String keyword) throws IOException {
        return contentService.parseContent(keyword);
    }
    // 查询数据
    @GetMapping("/searchJdPage/{keyword}/{pageNo}/{pageSize}")
    public List<Map<String, Object>> searchJdPage(@PathVariable("keyword") String keyword,
                                                  @PathVariable("pageNo") int pageNo,
                                                  @PathVariable("pageSize") int pageSize) throws IOException {
        return contentService.serachPage(keyword, pageNo, pageSize);
    }

```



