> 本文记录使用hight level rest client 在springboot中集成elasticsearch，而elasticsearch官方也推荐使用`hight level rest client`来操作es。spring-data-elasticsearch虽然也比较方便集成，但是由于其不是官方直接维护，所以存在版本严重落后现象。

#### 1.安装elasticsearch

##### 1.1 环境配置

- 安装docker
- 安装docker-compose
- 配置系统参数

```
# 设置内核参数
sysctl -w vm.max_map_count=262144
# 生效设置
sysctl -p
# 重启 docker，让内核参数对docker服务生效
systemctl restart docker
```

##### 1.2 安装

使用docker-compose安装elasticsearch7.1.1以及kibana7.1.1

```yml
version: "3"
services:
  elasticsearch:
    image: "elasticsearch:7.1.1"
    container_name: "elasticsearch"
    restart: "always"
    volumes:
      - "elasticsearch:/usr/share/elasticsearch"
    #vim /etc/sysctl.conf
    #vm.max_map_count=262144
    #sysctl -w vm.max_map_count=262144
    #sysctl -p
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.type=single-node
    networks:
      - "elk"
    ports:
      - "9200:9200"
      - "9300:9300"
  kibana:
    image: "kibana:7.1.1"
    container_name: "kibana"
    restart: "always"
    depends_on:
      - elasticsearch
    volumes:
      - "kibana:/usr/share/kibana"
    networks:
      - "elk"
    ports:
      - "5601:5601"
networks:
  elk:

volumes:
  elasticsearch:
  kibana:
```

```
运行docker容器
docker-compose up -d --build
查看日志
docker-compose logs -f 
```

##### 1.3 配置

安装完成之后开启es跨域，这样外部head插件或者kibana就可访问到

```yml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
http.cors.enabled: true
http.cors.allow-origin: "*"
```

配置完成之后重启elasticsearch和kibana

#### 2.集成springboot

##### 2.1引入依赖

```xml
<!-- es依赖 -->
<dependency>
      <groupId>org.elasticsearch</groupId>
      <artifactId>elasticsearch</artifactId>
      <version>7.3.0</version>
    </dependency>
    <dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>elasticsearch-rest-client</artifactId>
      <version>7.3.0</version>
    </dependency>
<dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>elasticsearch-rest-high-level-client</artifactId>
      <version>7.3.0</version>
      <exclusions>
        <exclusion>
          <groupId>org.elasticsearch.client</groupId>
          <artifactId>elasticsearch-rest-client</artifactId>
        </exclusion>
        <exclusion>
          <groupId>org.elasticsearch</groupId>
          <artifactId>elasticsearch</artifactId>
        </exclusion>
      </exclusions>
    </dependency>

	<!-- 配置解析处理 -->
	<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-configuration-processor</artifactId>
      <optional>true</optional>
    </dependency>
	<!-- lombok -->
	<dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
    </dependency>
	<!-- hutool工具 -->
	<dependency>
      <groupId>cn.hutool</groupId>
      <artifactId>hutool-all</artifactId>
    </dependency>
```

##### 2.2 配置

- 属性配置类

```java
/**
 * @author haopeng
 */
@ConfigurationProperties(prefix = "elasticsearch")
@Data
public class ElasticsearchProperties {

    /**
     * 请求协议
     */
    private String schema = "http";

    /**
     * 集群名称
     */
    private String clusterName = "elasticsearch";

    /**
     * 集群节点
     */
    private List<String> clusterNodes = new ArrayList<>();

    /**
     * 连接超时时间(毫秒)
     */
    private Integer connectTimeout = 1000;

    /**
     * socket 超时时间
     */
    private Integer socketTimeout = 30000;

    /**
     * 连接请求超时时间
     */
    private Integer connectionRequestTimeout = 500;

    /**
     * 每个路由的最大连接数量
     */
    private Integer maxConnectPerRoute = 10;

    /**
     * 最大连接总数量
     */
    private Integer maxConnectTotal = 30;


    /**
     * 索引配置信息
     */
    private Index index = new Index();

    /**
     * 认证账户
     */
    private Account account = new Account();


    /**
     * 索引配置信息
     */
    @Data
    public static class Index {

        /**
         * 分片数量
         */
        private Integer numberOfShards = 3;

        /**
         * 副本数量
         */
        private Integer numberOfReplicas = 0;

    }

    /**
     * 认证账户
     */
    @Data
    public static class Account {

        /**
         * 认证用户
         */
        private String username;

        /**
         * 认证密码
         */
        private String password;

    }

}

```



- 客户端配置类

```xml
/**
 * @author haopeng
 */
@Configuration
@EnableConfigurationProperties(ElasticsearchProperties.class)
public class ElasticsearchConfig {

    @Autowired
    private ElasticsearchProperties elasticsearchProperties;

    private List<HttpHost> httpHosts = new ArrayList<>();

    @Bean
    @ConditionalOnMissingBean
    public RestHighLevelClient restHighLevelClient() {
        List<String> clusterNodes = elasticsearchProperties.getClusterNodes();
        if (clusterNodes.isEmpty()) {
            throw new RuntimeException("集群节点不允许为空");
        }
        clusterNodes.forEach(node -> {
            try {
                String[] parts = StringUtils.split(node, ":");
                Assert.notNull(parts, "Must defined");
                Assert.state(parts.length == 2, "Must be defined as 'host:port'");
                httpHosts.add(new HttpHost(parts[0], Integer.parseInt(parts[1]), elasticsearchProperties.getSchema()));
            } catch (Exception e) {
                throw new IllegalStateException("Invalid ES nodes " + "property '" + node + "'", e);
            }
        });
        RestClientBuilder builder = RestClient.builder(httpHosts.toArray(new HttpHost[0]));
        return getRestHighLevelClient(builder, elasticsearchProperties);
    }

    /**
     * get restHistLevelClient
     * @return
     */
    private static RestHighLevelClient getRestHighLevelClient(RestClientBuilder builder, ElasticsearchProperties elasticsearchProperties) {
        // Callback used the default {@link RequestConfig} being set to the {@link CloseableHttpClient}
        builder.setRequestConfigCallback(requestConfigBuilder -> {
            requestConfigBuilder.setConnectTimeout(elasticsearchProperties.getConnectTimeout());
            requestConfigBuilder.setSocketTimeout(elasticsearchProperties.getSocketTimeout());
            requestConfigBuilder.setConnectionRequestTimeout(elasticsearchProperties.getConnectionRequestTimeout());
            return requestConfigBuilder;
        });
        // Callback used to customize the {@link CloseableHttpClient} instance used by a {@link RestClient} instance.
        builder.setHttpClientConfigCallback(httpClientBuilder -> {
            httpClientBuilder.setMaxConnTotal(elasticsearchProperties.getMaxConnectTotal());
            httpClientBuilder.setMaxConnPerRoute(elasticsearchProperties.getMaxConnectPerRoute());
            return httpClientBuilder;
        });
        // Callback used the basic credential auth
        ElasticsearchProperties.Account account = elasticsearchProperties.getAccount();
        if (!StringUtils.isEmpty(account.getUsername()) && !StringUtils.isEmpty(account.getUsername())) {
            final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();

            credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(account.getUsername(), account.getPassword()));
        }
        return new RestHighLevelClient(builder);
    }
}

```



- 实体类（LOL游戏人物）

```java
/**
 * 英雄联盟游戏人物
 * @author haopeng
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Lol implements Serializable {
    private Long id;
    /**
     * 英雄游戏名字
     */
    private String name;
    /**
     * 英雄名字
     */
    private String realName;
    /**
     * 英雄描述信息
     */
    private String desc;
}
```

##### 2.3 编写接口与实现类

- ES操作接口

```java
public interface IEsService {

    /**
     * 创建索引库
     */
    void createIndexRequest(String index);

    /**
     * 删除索引库
     */
    void deleteIndexRequest(String index);

    /**
     * 更新索引文档
     */
    void updateRequest(String index, String id, Object object);

    /**
     * 新增索引文档
     */
    void insertRequest(String index, String id, Object object);

    /**
     * 删除索引文档
     */
    void deleteRequest(String index, String id);
}
```

- 接口实现类

```java
@Slf4j
public class EsserviceImpl implements IEsService {

    @Autowired
    public RestHighLevelClient client;

    protected static final RequestOptions COMMON_OPTIONS;

    static {
        RequestOptions.Builder builder = RequestOptions.DEFAULT.toBuilder();
        // 默认缓冲限制为100MB，此处修改为30MB。
        builder.setHttpAsyncResponseConsumerFactory(new HttpAsyncResponseConsumerFactory.HeapBufferedResponseConsumerFactory(30 * 1024 * 1024));
        COMMON_OPTIONS = builder.build();
    }

    @Override
    public void createIndexRequest(String index) {
        CreateIndexRequest createIndexRequest = new CreateIndexRequest(index)
                .settings(Settings.builder().put("index.number_of_shards", 3).put("index.number_of_replicas", 0));
        try {
            CreateIndexResponse response = client.indices().create(createIndexRequest, COMMON_OPTIONS);
            log.info(" 所有节点确认响应 : {}", response.isAcknowledged());
            log.info(" 所有分片的复制未超时 :{}", response.isShardsAcknowledged());
        } catch (IOException e) {
            log.error("创建索引库【{}】失败", index, e);
        }
    }

    @Override
    public void deleteIndexRequest(String index) {
        DeleteIndexRequest request = new DeleteIndexRequest(index);
        try {
            AcknowledgedResponse response = client.indices().delete(request, COMMON_OPTIONS);
            System.out.println(response.isAcknowledged());
        } catch (IOException e) {
            log.error("删除索引库【{}】失败", index, e);
        }
    }

    @Override
    public void updateRequest(String index, String id, Object object) {
        UpdateRequest updateRequest = new UpdateRequest(index, id);
        updateRequest.doc(BeanUtil.beanToMap(object), XContentType.JSON);
        try {
            client.update(updateRequest, COMMON_OPTIONS);
        } catch (IOException e) {
            log.error("更新索引文档 {" + index + "} 数据 {" + object + "} 失败", e);
        }
    }

    @Override
    public void insertRequest(String index, String id, Object object) {
        IndexRequest indexRequest = new IndexRequest(index).id(id).source(BeanUtil.beanToMap(object), XContentType.JSON);
        try {
            client.index(indexRequest, COMMON_OPTIONS);
        } catch (IOException e) {
            log.error("创建索引文档 {" + index + "} 数据 {" + object + "} 失败", e);
        }

    }

    @Override
    public void deleteRequest(String index, String id) {
        DeleteRequest deleteRequest = new DeleteRequest(index, id);
        try {
            client.delete(deleteRequest, COMMON_OPTIONS);
        } catch (IOException e) {
            log.error("删除索引文档 {" + index + "} 数据id {" + id + "} 失败", e);
        }
    }
}
```

- LolService接口实现

```java
/**
 * @author haopeng
 */
@Service
@Slf4j
public class LolService extends EsserviceImpl {

    public void insertBach(String index, List<Lol> list) {
        if (list.isEmpty()) {
            log.warn("bach insert index but list is empty ...");
            return;
        }
        list.forEach((lol)->{
            super.insertRequest(index, lol.getId().toString(), lol);
        });
    }
    
    public List<Lol> searchList(String index) {
        SearchResponse searchResponse = search(index);
        SearchHit[] hits = searchResponse.getHits().getHits();
        List<Lol> lolList = new ArrayList<>();
        Arrays.stream(hits).forEach(hit -> {
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            Lol lol = BeanUtil.mapToBean(sourceAsMap, Lol.class, true);
            lolList.add(lol);
        });
        return lolList;
    }

    protected SearchResponse search(String index) {

        SearchRequest searchRequest = new SearchRequest(index);
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder.query(QueryBuilders.matchAllQuery());
        //bool符合查询
        //BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder()
        //        .filter(QueryBuilders.matchQuery("name", "盖伦"))
        //        .must(QueryBuilders.matchQuery("desc", "部落"))
        //        .should(QueryBuilders.matchQuery("realName", "光辉"));

        //分页
        //searchSourceBuilder.from(1).size(2);
        // 排序
        //searchSourceBuilder.sort("", SortOrder.DESC);

        ////误拼写时的fuzzy模糊搜索方法 2表示允许的误差字符数
        //QueryBuilders.fuzzyQuery("title", "ceshi").fuzziness(Fuzziness.build("2"));
        searchRequest.source(searchSourceBuilder);
        System.out.println(searchSourceBuilder.toString());
        System.out.println(JSONUtil.parseObj(searchSourceBuilder.toString()).toStringPretty());
        SearchResponse searchResponse = null;
        try {
            searchResponse = client.search(searchRequest, COMMON_OPTIONS);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return searchResponse;
    }

}
```

##### 2.4 单元测试

~~~java
@SpringBootTest
@RunWith(SpringRunner.class)
public class EsTest {

    public static final String INDEX_NAME = "lol";

    @Autowired
    private LolService lolService;

    @Test
    public void createIndex() {
        lolService.createIndexRequest(INDEX_NAME);
    }

    @Test
    public void deleteIndex() {
        lolService.deleteIndexRequest(INDEX_NAME);
    }


    @Test
    public void insertTest() {
        List<Lol> list = new ArrayList<>();
        list.add(Lol.builder().id(1L).name("德玛西亚之力").realName("盖伦").desc("作为一名自豪而高贵的勇士，盖伦将自己当做无畏先锋中的普通一员参与战斗。他既受到同袍手足的爱戴，也受到敌人对手的尊敬--尤其作为尊贵的冕卫家族的子嗣，他被委以重任，守卫德玛西亚的疆土和理想。他身披抵御魔法的重甲，手持阔剑，时刻准备着用正义的钢铁风暴在战场上正面迎战一切操纵魔法的狂人。").build());
        list.add(Lol.builder().id(2L).name("疾风剑豪").realName("亚索(快乐风男)").desc("亚索是一个百折不屈的艾欧尼亚人，也是一名身手敏捷的御风剑客。这位生性自负的年轻人，被误认为杀害长老的凶手--由于无法证明自己的清白，他出于自卫而杀死了自己的哥哥。虽然长老死亡的真相已然大白，亚索还是无法原谅自己的所作所为。他在家园的土地上流浪，只有疾风指引着他的剑刃。").build());
        list.add(Lol.builder().id(3L).name("魂锁典狱长").realName("锤石").desc("暴虐又狡猾的锤石是一个来自暗影岛的亡灵，野心勃勃、不知疲倦。他曾经是无数奥秘的看守，寻找着超越生死的力量，而现在他则使用自己独创的钻心痛苦缓慢地折磨并击溃其他人，以此作为自己存在下去的手段。被他迫害的人需要承受远超死亡的痛苦，因为锤石会让他们的灵魂也饱尝剧痛，将他们的灵魂囚禁在自己的灯笼中，经受永世的折磨。").build());
        list.add(Lol.builder().id(4L).name("圣枪游侠").realName("卢锡安").desc("曾担光明哨兵的卢锡安是一位冷酷的死灵猎人。他用一对圣物手枪无情地追猎并灭绝不死亡灵。为亡妻复仇的意念吞噬了他，让他无止无休。除非消灭锤石，那个手握她灵魂的恶鬼。冷酷而且决绝的卢锡安不允许任何东西挡住自己的复仇之路。如果有什么人或者什么东西愚蠢到敢挑衅他的原则，就必将接受压倒性的神圣枪火狂轰滥炸。").build());
        list.add(Lol.builder().id(5L).name("法外狂徒格雷福斯").realName("格雷福斯").desc("马尔科姆.格雷福斯是有名的佣兵、赌徒和窃贼，凡是他到过的城邦或帝国，都在通缉悬赏他的人头。虽然他脾气暴躁，但却非常讲究黑道的义气，他的双管散弹枪“命运”就经常用来纠正背信弃义之事。几年前他和老搭档崔斯特.菲特冰释前嫌，如今二人一同在比尔吉沃特的地下黑道纷争中再次如鱼得水。").build());
        list.add(Lol.builder().id(6L).name("光辉女郎").realName("拉克丝").desc("拉克珊娜.冕卫出身自德玛西亚，一个将魔法视为禁忌的封闭国度。只要一提起魔法，人们总是带着恐惧和怀疑。所以拥有折光之力的她，在童年的成长过程中始终担心被人发现进而遭到放逐，一直强迫自己隐瞒力量，以此保住家族的贵族地位。虽然如此，拉克丝的乐观和顽强让她学会拥抱自己独特的天赋，现在的她正在秘密地运用自己的能力为祖国效力。").build());
        list.add(Lol.builder().id(7L).name("发条魔灵").realName("奥莉安娜").desc("奥莉安娜曾是一个拥有血肉之躯的好奇女孩，而现在则是全身上下部由发条和齿轮构成的科技奇观。祖安下层地区的一次事故间接导致了她身染重病，日渐衰竭的身体必须替换为精密的人造器官，一个接一个，直到全身上下再也没有人类的肉体。她给自己制作了一枚奇妙的黄铜球体，既是伙伴，也是保镖。如今她已经可以自由地探索壮观的皮尔特沃夫，以及更遥远的地方。").build());

        lolService.insertBach(INDEX_NAME, list);
    }

    @Test
    public void updateTest() {
        Lol lol = Lol.builder().id(6L).name("殇之木乃伊").realName("阿木木").desc("或许阿木木是英雄联盟世界里最古老的保卫者英雄之一，他对加入联盟前的生活仍一无所知。阿木木唯一记得的是自己在Shuima沙漠的一座金字塔内独自醒来。他全身缠着裹尸布，感受不到自己的心跳。此外，他感到一股强大而莫名的悲伤；他知道他失去了亲人，虽然他已不记得他们是谁。阿木木跪下来，在绷带内哭泣。不论做什么，似乎他都无法阻止眼泪或悲伤。最后他站起来在这个世界上游荡，并进入了联盟").build();
        lolService.updateRequest(INDEX_NAME, lol.getId().toString(), lol);
    }

    @Test
    public void deleteTest() {
        lolService.deleteRequest(INDEX_NAME, "1");
    }

    /**
     * 测试查询
     */
    @Test
    public void searchListTest() {
        List<Lol> personList = lolService.searchList(INDEX_NAME);
        System.out.println(personList);
    }
}
~~~

- 可以通过`kibana`中`DevTools`进行查询验证结果

```
//查询所有
GET /lol/_search
{
  "query": {
   "match_all": {}
  }
}
//match查询
GET /lol/_search
{
  "query": {
  "match": {
    "name": "魔灵"
  }
  }
}
//term查询
POST /lol/_search
{
  "query": {
   "term": {
     "_id": {
       "value": "7"
     }
   }
  }
}
```






![](https://upload-images.jianshu.io/upload_images/8387919-7cb2d30ef5f873be.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/8387919-0b1af5b5231c8f54.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/8387919-6f660ae540569aa3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

#### 参考资料：

- [Java High Level REST Client](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.x/java-rest-high.html)

- [复合查询](https://www.cnblogs.com/asker009/p/10179544.html)