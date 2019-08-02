---
layout:     post
title:      ElasticSearch在SpringBoot2.0中的使用
subtitle:   —— esJPA2.16 + mysql5.7 + kafka2.1.1 + elasticSearch5.6.16
date:       2019-08-02
author:     Zwx
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spring
    - ElasticSearch
---

----
## 前言：

   最近在学习全文本搜索引擎ElasticSearch，记录一下使用SpringBoot2.0整合es的过程：
   写了一个简单的接口，向mysql中插入数据的时候给kafka发送消息，消费者接收到消息后解析消息内容，并异步创建es索引，保存到es中。
   然后所有的查询接口都查es中的内容，达到更好的搜索效果，并减轻mysql压力。

----

## 项目结构：
![](http://pic.zwxzzz.top/QQ%E6%88%AA%E5%9B%BE20190802172722.png)

#### 项目依赖
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--spring-data-es-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <!--kafka-->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
            <version>2.2.1.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.44</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
```

#### 配置
```yaml
spring:
  data:
    elasticsearch:
      cluster-name: zwx
      cluster-nodes: xx:9300
  kafka:
    bootstrap-servers: xx:9092
    consumer:
      group-id: test-consumer-group
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://xx:3306/laozhangdb
    username: root
    password: xx
  jpa:
    hibernate:
      ddl-auto: update
      show-sql: true
  main:
    allow-bean-definition-overriding: true

server:
  port: 8080
```

#### es实体类
```java
/**
 * @program: es_demo
 * @description: 书
 * @author: Zwx
 * @create: 2019-07-24 15:58
 **/
@Data
@Document(indexName = "book",type = "doc",shards = 1,replicas = 0)
public class Book implements Serializable {
    @Id
    private Long id;
    /**
     * 不加Type的话 默认FieldType.Auto jpa也可以通过字段的值猜出它的类型
     * 这里也指定了中文分词 ik_smart ，具体安装方法见百度
     */
    @Field(type = FieldType.Text,analyzer = "ik_smart")
    private String name;

    public Book() {
    }

    public Book(Long id, String name) {
        this.id = id;
        this.name = name;
    }
}
```

#### dao层
```java
public interface EsBookRepository extends ElasticsearchRepository<Book,Long> {
    List<EsBook> findByIdBetween(Long a, Long b);

    List<EsBook> findByNameLike(String s);
}
```
 - 注意这里父类的参数ElasticsearchRepository<a,b>，a为上面的实体类，b为实体类中@Id注解下面的字段类型。
 - jpa自定义方法命名要遵循上一篇博客的规则
 
 
#### 单元测试类
```java

@RunWith(SpringRunner.class)
@SpringBootTest
public class EsDemoApplicationTests {

    @Autowired(required=false)
    private ElasticsearchTemplate esTemplate;

    @Autowired
    private MegacorpRepository repository;

    @Autowired
    private EsBookRepository bookRepository;

    /**
     * 新建索引
     */
    @Test
    public void createIndex() {
        esTemplate.createIndex(Book.class);
    }

    @Test
    public void deleteIndex(){
        esTemplate.deleteIndex(Book.class);
    }

    /**
     * 新增 & 修改 （按ID决定）
     */
    @Test
    public void save() {
//        Megacorp megacorp = new Megacorp(1L,"1","1","1",10L,"1");
//        repository.save(megacorp);
        Book book =new Book(2L,"java实战开发经典");
        bookRepository.save(book);
    }

    /**
     * 批量新增
     */
    @Test
    public void saveall() {
        List<Megacorp> megacorps = new ArrayList<>();
        megacorps.add(new Megacorp(2L,"2","1","1",1L,"1"));
        megacorps.add(new Megacorp(3L,"3","1","在实现一个字段模糊查询的时候",1L,"1"));
        megacorps.add(new Megacorp(4L,"2","1","在实现两个字段模糊查询的时候",1L,"1"));
        megacorps.add(new Megacorp(5L,"3","1","在实现三个字段模糊查询的时候",1L,"1"));
        megacorps.add(new Megacorp(6L,"2","1","在实现四个字段模糊查询的时候",1L,"1"));
        megacorps.add(new Megacorp(7L,"3","1","在实现一五个字段模糊查询的时候",1L,"1"));
        repository.saveAll(megacorps);
    }

    /**
     * 基本查询
     * 注意这里查询时候对应的实体类中要有一个无参的构造函数
     */
    @Test
    public void query() {
        Iterable<Megacorp> megacorps = repository.findAll();
        for (Megacorp c:megacorps){
            System.out.println(c);
        }
    }

    /**
     * Jpa自定义函数查询
     * 自定义方法具体命名规则见: http://www.zwxzzz.top/2019/07/23/SpringDataJPA%E8%87%AA%E5%AE%9A%E4%B9%89%E6%96%B9%E6%B3%95%E7%BA%A6%E5%AE%9A/
     */
    @Test
    public void queryBerween() {
        Iterable<Megacorp> megacorps = repository.findByIdBetween(2L,4L);
        for (Megacorp c:megacorps){
            System.out.println(c);
        }
    }

    @Test
    public void matchQuery(){
        // 构建查询条件
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        // 添加基本分词查询
        queryBuilder.withQuery(QueryBuilders.matchQuery("last_name", "2"));
//        termQuery:功能更强大，除了匹配字符串以外，还可以匹配
//        int/long/double/float/....
//        queryBuilder.withQuery(QueryBuilders.termQuery("last_name", "2"));
        // 搜索，获取结果
        Page<Megacorp> items = repository.search(queryBuilder.build());
        // 总条数
        long total = items.getTotalElements();
        System.out.println("total = " + total);
        for (Megacorp item : items) {
            System.out.println(item);
        }
    }

    /**
     * 布尔查询
     */
    @Test
    public void booleanQuery(){
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        queryBuilder.withQuery(
                QueryBuilders.boolQuery().must(QueryBuilders.matchQuery("age",10))
        );
        Page<Megacorp> megacorps = repository.search(queryBuilder.build());
        for (Megacorp m:megacorps){
            System.out.println(m);
        }
    }

    /**
     * 模糊查询  要注意 分词分完的结果
     */
    @Test
    public void fuzzyQuery(){
//        方法1
        NativeSearchQueryBuilder builder = new NativeSearchQueryBuilder();
        builder.withQuery(QueryBuilders.fuzzyQuery("name","java"));
        Page<Book> page = bookRepository.search(builder.build());
//        方法2
//        Iterable<Megacorp> page = repository.findByInterestsLike("模糊");
        for (Book m :page){
            System.out.println(m);
        }
    }

    /**
     * @Description: 分页查询
     * @Author:
     */
    @Test
    public void searchByPage(){
        // 构建查询条件
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        // 添加基本分词查询
        queryBuilder.withQuery(QueryBuilders.fuzzyQuery("interests", "一个"));
        // 分页：
        int page = 0;
        int size = 2;
        queryBuilder.withPageable(PageRequest.of(page,size));
        // 搜索，获取结果
        Page<Megacorp> items = repository.search(queryBuilder.build());
        // 总条数
        long total = items.getTotalElements();
        System.out.println("总条数 = " + total);
        // 总页数
        System.out.println("总页数 = " + items.getTotalPages());
        // 当前页
        System.out.println("当前页：" + items.getNumber());
        // 每页大小
        System.out.println("每页大小：" + items.getSize());
        for (Megacorp item : items) {
            System.out.println(item);
        }
    }

    /**
     * @Description:排序查询
     */
    @Test
    public void searchAndSort(){
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        queryBuilder.withQuery(QueryBuilders.termQuery("interests", "一"));
        // 排序
        queryBuilder.withSort(SortBuilders.fieldSort("id").order(SortOrder.DESC));
        // 搜索，获取结果
        Page<Megacorp> items = repository.search(queryBuilder.build());
        // 总条数
        long total = items.getTotalElements();
        System.out.println("总条数 = " + total);

        for (Megacorp item : items) {
            System.out.println(item);
        }
    }

    /**
     * @Description: 按照 字段 进行分组
     */
    @Test
    public void testAgg(){
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        // 不查询任何结果
        queryBuilder.withSourceFilter(new FetchSourceFilter(new String[]{""}, null));
        // 1、添加一个新的聚合，聚合类型为Megacorp，聚合名称为last_names，聚合字段为last_name
        queryBuilder.addAggregation(
                AggregationBuilders.terms("last_names").field("last_name"));
        // 2、查询,需要把结果强转为AggregatedPage类型
        AggregatedPage<Megacorp> aggPage = (AggregatedPage<Megacorp>) repository.search(queryBuilder.build());
        // 3、解析
        // 3.1、从结果中取出名为brands的那个聚合，
        // 因为是利用String类型字段来进行的Megacorp聚合，所以结果要强转为StringTerm类型
        StringTerms agg = (StringTerms) aggPage.getAggregation("last_names");
        // 3.2、获取桶
        List<StringTerms.Bucket> buckets = agg.getBuckets();
        // 3.3、遍历
        for (StringTerms.Bucket bucket : buckets) {
            // 3.4、获取桶中的key，即last_name
            System.out.println("Key:"+bucket.getKeyAsString());
            // 3.5、获取桶中的文档数量
            System.out.println("数量："+bucket.getDocCount());
        }
    }
}

```

- 对es的操作既可以使用esTemplate，也可以使用自定义方法，复杂查询时候可以用SpringJPA提供的NativeSearchQueryBuilder，这里具体的我也在深入学习中。

## kafka
- 我这里用的语法很简单：
- 发送消息：kafkaTemplate.send();
- 接受消息：@KafkaListener注解，然后接收ConsumerRecord<?,?>

生产者：
```java
/**
 * @program: es_demo
 * @description: 消息生产者
 * @author: Zwx
 * @create: 2019-07-30 16:34
 **/
@Component
@Slf4j
public class KafkaProducer {

    @Autowired
    private KafkaTemplate kafkaTemplate;

    public void sendMsg(String content){
        IndexMessage msg = new IndexMessage(1,null,"哈喽哈喽",0);
        log.info(msg.toString());
        kafkaTemplate.send(Constant.TOPIC, JSON.toJSONString(msg));
    }

    public void insertBook(Book book){
        IndexMessage msg = new IndexMessage(1,book,"新增",0);
        log.info(msg.toString());
        kafkaTemplate.send(Constant.TOPIC, JSON.toJSONString(msg));
    }
}
```

消费者：
```java
/**
 * @program: es_demo
 * @description:
 * @author: Zwx
 * @create: 2019-07-30 16:34
 **/
@Component
@Slf4j
public class KafkaConsumer {

    @Autowired
    private EsBookRepository bookRepository;

    @Autowired
    private ObjectMapper objectMapper;

    @KafkaListener(topics = Constant.TOPIC)
    public void consumer(ConsumerRecord<?,?> consumerRecord) throws IOException {
//        判断是否为null
        Optional<?> kafkaMessage = Optional.ofNullable(consumerRecord.value());
        log.info("收到消息————————————>"+kafkaMessage);
        if(kafkaMessage.isPresent()){
            Object message = kafkaMessage.get();
            log.info("消息" + message);
            IndexMessage indexMessage = objectMapper.readValue(message.toString(),IndexMessage.class);
            bookRepository.save(indexMessage.getBook());
        }
    }
}
```