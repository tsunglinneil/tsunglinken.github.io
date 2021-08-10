---
layout: post
title: Testcontainers整合測試實務
date: 2021-08-04 15:49:41 +0800
image: testcontainers-article-cover.png
tags:
  [
    docker,
    containers,
    testcontainers,
    kafka,
    redis,
    rdbms,
    nosql,
    integrationtest,
    springboot,
    整合測試,
  ]
---

整合測試在本地端進行往往會與實際部署環境有很大的落差，然而 Docker 容器化的技術使得環境建立的門檻降低。

這篇文章將從整合測試在實務上會遇到的問題，並根據現今軟體開發的趨勢，搭配 Kafka、Redis、RDBMS，說明如何運用容器化的技術解決實務上測試的困難點。

## 本篇文章中提及之各種測試名詞說明

- 單元測試 → 開發人員撰寫的單元測試
- 整合測試 → 開發人員撰寫的整合測試
- 手動整合測試 → 部署到共享環境中的使用者測試

若想暸解更多關於整合測試/單元測試，提供以下昕力大學資源參考

- [Spring Boot 的單元測試](https://www.tpisoftware.com/tpu/articleDetails/1256)
- [模擬測試框架 Mockito 介紹](https://www.tpisoftware.com/tpu/articleDetails/1294)

## 談談整合測試實務上的問題

<div class="row">
    <div class="col"><img src="{{site.baseurl}}/images/testcontainers/1.png" width="150" height="150" alt=""></div>
    <div class="col"><img src="{{site.baseurl}}/images/testcontainers/turn.png" width="150" height="150" alt=""></div>
    <div class="col"><img src="{{site.baseurl}}/images/testcontainers/2.png" width="150" height="150" alt=""></div>
</div>
如上圖可看到廣為人知的測試金字塔(此概念源自於*[Mike Cohn](https://www.mountaingoatsoftware.com/company/about-mike-cohn)* ，可查閱[這篇 BLOG 文章](https://www.mountaingoatsoftware.com/blog/the-forgotten-layer-of-the-test-automation-pyramid))，描述了各個層級的測試比重應該要是單元測試>整合測試>手動整合測試，然而實務上的狀況則通常相反，手動整合集成測試的比重最高 → 從金字塔變成了冰淇淋 ?

**整合測試遇到的問題大致如下：**

- 整合測試相較於單元測試，運行的時間和成本都較高。
- 環境建置複雜，需要在開發環境預先安裝建置好所需工具，例如: 資料庫等外部依賴。

藉由上述的問題，環境建置複雜度的成本是更主要的原因，因此可以理解為何許多情況下會優先考慮直接部署到 SIT/UAT 環境進行手動整合測試。

## 所以就繼續手動整合測試吧!?

我們先把目光轉移到幾個面向

- **微服務。**
- **分散式。**

在上述主流的架構當中，服務和存儲裝置不再只是集中在一個地方，而是能夠分散到在各地運作，以達到靈活、延展、高可用、資料隔離等等優點，但也衍生了一些問題

- 多個服務之間的介接測試成本高，需要部署多個服務後才能進行完整測試。
- 資料又該如何共享。

為了因應上述的架構，往往會需要搭配一些解決方案

- 例如你需要部署的服務變多了，需要導入 CI/CD 自動化你的部署流程，節省開發人員的工作，但相對自動部署頻率的提高，也使得測試人員可能更無法即時的去測試每次更新的正確性。
- 例如你可能還會需要搭建外部緩存機制、消息代理等等，也就是外部依賴數量增加，需要測試的環節也變多了，每次為了驗證這些外部依賴，又只依靠手動整合測試的情況下，可能只能增加部署的頻率。

尚未實施上述架構的團隊，可能先不用擔心這些問題，但相信一定有遇過下列情境

- 多團隊協作，在共享環境中部署、更新了共享資源，例如:資料庫、第三方套件等等，造成其中一方的功能無法正確被驗證，而程式可能根本就沒有異動。
- 開發測試共用資料庫，因為多人開發，資料或數據庫被修改，造成測試結果失敗或不準確。
- 搭配 H2 撰寫整合測試時，即使通過測試，當部署到線上環境時依然會出錯，主要原因和問題如下

  1.  SQL 語法不兼容。
  2.  需維護多個版本的 SQL 語法。
  3.  環境資料表落差。

不論哪種情境，都顯示出整合測試在開發者的環境中進行是有必要的，但如同前面提到的，環境建置的成本是個大問題。

## 容器化整合測試方案選擇

既然有容器化的技術，使用 Container 來進行整合測試會是一個選擇。

- [Docker-Compose](https://docs.docker.com/compose/)

  將多個服務建構成 Images 以 docker-compose up 的方式執行，但有一些侷限性。

  1.  每個服務的 PORT 是固定的，在環境上的配置造成了限制，且無法進行並行測試。
  2.  多個測試可能無法同時進行，因為也許在 A 情境下的測試資料不應留到 B 情境。
  3.  當有多個不同服務或版本需要被測試時，配置將會是個問題。
  4.  每次測試前後都需要自行啟動/關閉 Container。

- [Maven Plugin - docker-maven-plugin](https://github.com/fabric8io/docker-maven-plugin)

  若想達到自動化測試，Maven 的配置方案也許可以考慮，但也有上述提到的部分問題。

- [Docker API](https://docs.docker.com/engine/api/v1.40/#)

  Docker 提供了 REST API，因此若是可以自行使用這些 API，將可使得整合測試的設計更靈活。

接著進入本篇重點，結合 JVM 與 Container 來實現整合測試

## 介紹 TestContainers

擷取[開源專案](https://github.com/testcontainers/testcontainers-java)上針對 TestContainers 的說明

`”Testcontainers is a Java 8 library that supports JUnit tests, providing lightweight, throwaway instances of common databases, Selenium web browsers, or anything else that can run in a Docker container.”`

初步可以得知，Testcontainers 是 Java8 的類庫，採用容器化的技術，能夠各自獨立運行並使用與 Production 相同版本的外部依賴，且 Container 能運行在各個不同平台的特性，減少了環境搭建的複雜度。

TestContainer 能夠使用任何具有 Docker Image 的外部依賴項目，例如: 資料庫、Web 瀏覽器工具、消息代理、網頁伺服器等等，同時也支援 JVM 的測試框架，例如: JUnit，另外還支持各種語言的版本，目前 Java 的版本是比較完整的（[點擊查閱支援項目清單](https://github.com/testcontainers)）。

## 應用場景

- **資料庫數據存取層的整合測試**

  Example: 任何容器化的資料庫類型，MySQL、Postgrest...

- **外部依賴項目的整合測試**

  Example: LDAP、Redis、Kafka、Micro Service、Nginx...

- **自動化 UI 整合測試**

  Example: Selenium browser

- **自動化整合測試**

  Example: dind-drone-plugin

## 導入團隊前的建議事項

- 具有 Docker 的概念及操作經驗將可幫助理解其運作。
- 確認並列出需要被測試的外部依賴項目，一開始先聚焦確認好測試的目的和方向會省下許多時間。

## 實際測試範例說明

### **▶︎ 專案結構簡述**

分別以 Kafka、Redis、RDBMS(MSSQL)進行，使用 Spring Boot 2 及 Junit 5，並引用前述依賴(Kafka、Redis、MSSQL)撰寫 Production Code 或配置，再引用 Testcontainers Spring Boot 的依賴撰寫測試。

**• Maven 依賴**

```markdown
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
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

<!-- test - junit 5 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <version>5.4.2</version>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <version>5.4.2</version>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-engine</artifactId>
    <version>5.4.2</version>
    <scope>test</scope>
</dependency>
```

### **▶︎** **實際演練-Kafka**

建置 Spring Boot 2 專案，並引用 Kafka 依賴，建立 Producer 發送消息、Consumer 訂閱 Topic 取得並回應訊息，這裡在本地環境不去安裝及建立 Kafka，也不使用 Embedded Kafka，而是使用 TestContainers 協助建立 Kafka Container 進行整合測試。

**• Maven 依賴**

```markdown
<!-- kafka streams -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
</dependency>

<!-- kafka spring -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>

<!-- test - testcontainers - kafka -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <version>1.14.3</version>
    <scope>test</scope>
</dependency>

<!-- test - kafka test util -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka-test</artifactId>
    <scope>test</scope>
</dependency>
```

**說明：**

spring-kafka-test 依賴，可以讓我們輕易的去驗證 Producer＆Consumer 的資料狀態。

**• Kafka Producer 實作**

```java
@Component
@ConditionalOnProperty(name = "enable.kafka", havingValue="true")
public class KafkaSenderService {
    private static Logger logger = LoggerFactory.getLogger(DatabaseInitialService.class);

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @EventListener(ApplicationStartedEvent.class)
    public void send() {
        kafkaTemplate.send("testcontainers", "kafka", "TestcontainersKafka").addCallback(result -> {
            if (result != null) {
                RecordMetadata recordMetadata = result.getRecordMetadata();
                logger.info("producer send data to {}, {}, {}", recordMetadata.topic(), recordMetadata.partition(),
                        recordMetadata.offset());
            }
        }, ex -> {
            logger.error("something wrong...", ex);
        });

        kafkaTemplate.flush();
    }
}
```

**說明：**

利用 KafkaTemplate 來幫助我們更容易的建立 Topic 以及欲寫入的資料

10: 寫入 key/value "kafka"/"TestcontainersKafka" 給 Kafka Topic "testcontainers"，並 log 印出資訊。

**• 建立測試**

```java
@SpringBootTest
class KafkaTest {
    private static Logger logger = LoggerFactory.getLogger(KafkaTest.class);

    static KafkaContainer kafkaContainer = new KafkaContainer();

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        kafkaContainer.start();
        registry.add("spring.kafka.properties.bootstrap.servers", kafkaContainer::getBootstrapServers);
        registry.add("spring.kafka.consumer.group-id", () -> "testcontainersapp");
        registry.add("spring.kafka.consumer.properties.auto.offset.reset", () -> "earliest");
    }

    @Autowired
    private KafkaProperties properties;

    @Test
    public void testKafkaProducerSendDataAndConsumerReceiveData() {
        final Consumer<String, String>[] consumer = new Consumer[]{createConsumer("testcontainers")};

        String actual = "";
        while (true) {
            ConsumerRecords<String, String> records = KafkaTestUtils.getRecords(consumer[0], 10000);
            if (records.isEmpty()) {
                break;
            }
            for (ConsumerRecord<String, String> record : records) {
                actual = record.value();
            }
        }

        assertEquals("TestcontainersKafka", actual);
    }

    private Consumer<String, String> createConsumer(String topicName) {
        Consumer<String, String>
                consumer = new DefaultKafkaConsumerFactory<>(properties.buildConsumerProperties(), StringDeserializer::new,
                        StringDeserializer::new).createConsumer();

        consumer.subscribe(Collections.singletonList(topicName));
        return consumer;
    }
}
```

**說明：**

5: 實例化 Kafka Container。

7-13: 啟動 Kafka Container 並動態指定設定檔參數。

18-34: 利用 Kafka Consumer API 取得訂閱的 Topic "testcontainers"資料。

36-43: 實例化 Kafka Consumer 並訂閱 Topic "testcontainers。

**• 執行測試**

**[啟動容器]**

<div><img src="{{site.baseurl}}/images/testcontainers/kafka/startup.png" width="800" height="150" alt=""></div>

**[測試結果]**

<div><img src="{{site.baseurl}}/images/testcontainers/kafka/run-test.png" width="800" height="150" alt=""></div>

### **▶︎** **實際演練-RDBMS(MSSQL)**

建置 Spring Boot 2 專案，並引用 MSSQL 依賴，這裡在本地環境不去安裝及建立 MSSQL，而是使用 TestContainers 協助建立 MSSQL Container 進行數據層的整合測試。

**• Maven 依賴**

```markdown
<!-- mssql server -->
<dependency>
    <groupId>com.microsoft.sqlserver</groupId>
    <artifactId>mssql-jdbc</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- test - testcontainers - mssql server -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mssqlserver</artifactId>
    <version>1.14.3</version>
    <scope>test</scope>
</dependency>
```

**• Model & Repository (使用 Spring Data JPA)**

```java
/** Model */
@Entity
@Table(name = "book_category", schema = "dbo")
public class BookCategory {
    private int bookSeq;
    private String categoryName;
    private int sort;
    private String suspend;

    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    @Column(name = "book_seq")
    public int getBookSeq() {
        return bookSeq;
    }

    public void setBookSeq(int bookSeq) {
        this.bookSeq = bookSeq;
    }

    @Basic
    @Column(name = "category_name")
    public String getCategoryName() {
        return categoryName;
    }

    public void setCategoryName(String categoryName) {
        this.categoryName = categoryName;
    }

    @Basic
    @Column(name = "sort")
    public int getSort() {
        return sort;
    }

    public void setSort(int sort) {
        this.sort = sort;
    }

    @Basic
    @Column(name = "suspend")
    public String getSuspend() {
        return suspend;
    }

    public void setSuspend(String suspend) {
        this.suspend = suspend;
    }

    public static BookCategory createBookCategory(String categoryName, int sort) {
        BookCategory bookCategory = new BookCategory();
        bookCategory.setCategoryName(categoryName);
        bookCategory.setSort(sort);
        bookCategory.setSuspend("N");
        return bookCategory;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        BookCategory that = (BookCategory) o;
        return bookSeq == that.bookSeq &&
            sort == that.sort &&
            Objects.equals(categoryName, that.categoryName) &&
            Objects.equals(suspend, that.suspend);
    }

    @Override
    public int hashCode() {
        return Objects.hash(bookSeq, categoryName, sort, suspend);
    }
}

/** Repository */
public interface BookCategoryRepository extends JpaRepository<BookCategory, Integer> {
}
```

**說明：**

建立 Entity 物件及 Spring Data JPA Repository。

**• 初始化資料表**

```sql
SET ANSI_NULLS ON

SET QUOTED_IDENTIFIER ON

SET ANSI_PADDING ON

CREATE TABLE book_category (
  book_seq int NOT NULL IDENTITY(1,1),
  category_name nvarchar(100) NOT NULL,
  sort int NOT NULL,
  suspend char(1) NOT NULL CONSTRAINT DF_book_category_suspend DEFAULT ('N'),
  CONSTRAINT PK_book_category PRIMARY KEY CLUSTERED (
    book_seq ASC
  ),
);

SET ANSI_PADDING OFF

EXEC sys.sp_addextendedproperty @name=N'MS_Description', @value=N'分類', @level0type=N'SCHEMA', @level0name=N'dbo', @level1type=N'TABLE', @level1name=N'book_category'

EXEC sys.sp_addextendedproperty @name=N'MS_Description', @value=N'流水號', @level0type=N'SCHEMA', @level0name=N'dbo', @level1type=N'TABLE', @level1name=N'book_category', @level2type=N'COLUMN', @level2name=N'book_seq'

EXEC sys.sp_addextendedproperty @name=N'MS_Description', @value=N'排序(1-99)', @level0type=N'SCHEMA', @level0name=N'dbo', @level1type=N'TABLE', @level1name=N'book_category', @level2type=N'COLUMN', @level2name=N'sort'

EXEC sys.sp_addextendedproperty @name=N'MS_Description', @value=N'停用否', @level0type=N'SCHEMA', @level0name=N'dbo', @level1type=N'TABLE', @level1name=N'book_category', @level2type=N'COLUMN', @level2name=N'suspend'

INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'文學小說', 1, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'商業理財', 2, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'藝術設計', 3, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'人文史地', 4, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'社會科學', 5, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'自然科普', 6, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'心理勵志', 7, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'醫療保健', 8, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'飲食', 9, 'N');
INSERT INTO dbo.book_category (category_name, sort, suspend) VALUES (N'生活風格', 10, 'N');
```

**說明：**

本範例之檔案放置於 src/main/resources/....

**• 建立測試**

```java
@SpringBootTest
public class DatabaseTest {
    private static Logger logger = LoggerFactory.getLogger(DatabaseTest.class);

    static MSSQLServerContainer mssqlserver = (MSSQLServerContainer) new MSSQLServerContainer()
        .withInitScript("doc/ddl.sql");

    @DynamicPropertySource
    static void mssqlProperties(DynamicPropertyRegistry registry) {
        mssqlserver.start();
        registry.add("spring.datasource.driver-class-name", mssqlserver::getDriverClassName);
        registry.add("spring.datasource.url", () -> mssqlserver.getJdbcUrl());
        registry.add("spring.datasource.username", mssqlserver::getUsername);
        registry.add("spring.datasource.password", mssqlserver::getPassword);
    }

    @Autowired
    private BookCategoryRepository bookCategoryRepository;

    @Test
    void testBookCategoryListSizeIs10() {
        List<BookCategory> bookCategoryList = bookCategoryRepository.findAll();
        assertEquals(10, bookCategoryList.size());
    }
}
```

**說明：**

5: 實例化 MSSQL SERVER Containers 並指定初始化資料庫的 Script 檔案，本範例會建立 book_category 資料表並寫入 10 筆資料。

8-15: 啟動 MSSQL SERVER Container，並動態指定設定檔參數。

20-24: 驗證可否從 book_category 資料表取出 10 筆資料。

**• 執行測試**

**[啟動容器]**

<div><img src="{{site.baseurl}}/images/testcontainers/mssql/startup.png" width="800" height="150" alt=""></div>

**[測試結果]**

<div><img src="{{site.baseurl}}/images/testcontainers/mssql/run-test.png" width="800" height="150" alt=""></div>

## **▶︎** **實際演練-Redis**

建置 Spring Boot 2 專案，並引用 Redis 依賴，這裡在本地環境不去安裝及建立 Redis，而是使用 TestContainers 協助建立 Redis Container 進行整合測試。

**• Maven 依賴**

```markdown
<!-- spring data redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- test - testcontainers -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.14.3</version>
    <scope>test</scope>
</dependency>
```

**• Redis 配置**

```java
@Configuration
public class RedisConfiguration {
    @Bean
    public RedisTemplate redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate redisTemplate = new RedisTemplate();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(connectionFactory);
        return redisTemplate;
    }
}
```

**說明：**

配置 RedisTemplate 並指定實作之 Serializer（也可依據需求使用客製化的 Serializer），當使用 RedisTemplate 將資料寫入 Redis 時，會根據指定 Serializer 進行處理，如此一來則無需在每個物件中實作序列化。

**• 建立測試**

```java
@SpringBootTest
public class RedisTest {
    private static Logger logger = LoggerFactory.getLogger(RedisTest.class);

    static GenericContainer redis = new GenericContainer("redis:5.0.5")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        redis.start();
        registry.add("spring.redis.host", redis::getContainerIpAddress);
        registry.add("spring.redis.port", redis::getFirstMappedPort);
    }

    @Autowired
    private RedisTemplate redisTemplate;

    @Test
    public void testSetAndGetWithString() {
        redisTemplate.opsForValue().set("k1", "v1");
        assertEquals("v1", redisTemplate.opsForValue().get("k1"));
    }

    @Test
    public void testSetAndGetWithObject() {
        BookCategory bookCategory = BookCategory.createBookCategory("生活風格", 10);
        redisTemplate.opsForValue().set("k2", bookCategory);
        String categoryName = ((BookCategory) redisTemplate.opsForValue().get("k2")).getCategoryName();
        assertEquals("生活風格", categoryName);
    }

    @After
    public void destory() {
        redis.stop();
    }
}
```

**說明：**

5: 實例化 Redis Container，這裡使用的是 GenericContainer，可以透過這個物件傳入客製化的 Image Name，增加測試容器的彈性。

8-13: 啟動 Redis Container，並動態指定設定檔參數。

18-22: 驗證從 Redis 寫入/取出字串資料的正確性。

24-30: 驗證從 Redis 寫入/取出物件資料的正確性。

**• 執行測試**

**[啟動容器]**

<div><img src="{{site.baseurl}}/images/testcontainers/redis/startup.png" width="800" height="150" alt=""></div>

**[測試結果-1]**

<div><img src="{{site.baseurl}}/images/testcontainers/redis/run-test1.png" width="800" height="150" alt=""></div>

**[測試結果-2]**

<div><img src="{{site.baseurl}}/images/testcontainers/redis/run-test2.png" width="800" height="150" alt=""></div>

# **實際演練心得**

單單就 Kafka、Redis、RDBMS 的情境分別實作的過程來說，透過 TestContainers 簡化了許多步驟，也無端口衝突的問題，當然可以想像在較為複雜的專案情境中一定會遇到一些不可預期的狀況。

**容器化的整合測試方案並非完全沒有缺點**，實際操作發現以下問題：

1. 每個測試都會啟用 Docker Container，也因此**花費時間較長**（自動部署運行自動化測試花費時間亦會有此問題），這部分則可以採用**並行化測試**來解決。

2. 需要進行**額外的配置工作**，像是針對初始化資料庫，本篇文章使用的是 MSSQL，而 Testcontainer 的 MSSQL 的解析實作上會擋掉某些語句，因此就必須要花費一些時間測試資料初始化的 Script。

然而整體來說，相較於每次都要部署到測試環境進行驗證，即便是使用自動化部署，部署上去版本難免還是可能發生不可預期狀況（例如推送失敗、合併問題等等），因此容器化整合測試我覺得是值得嘗試的一個解決方案。

有任何問題，歡迎聯繫討論，謝謝~!

---

References:

- **[官方文檔](https://www.testcontainers.org/)**
- **[好文章，說明了資料庫整合測試的問題以及為何我們需要使用 TestContainer](https://reflectoring.io/spring-boot-flyway-testcontainers/)**
- **[好資源，提供全面的範例及使用說明，讓你知道在不同情境下如何使用 Testcontainer](https://github.com/testcontainers/testcontainers-spring-boot)**
- **[好範例，Spring Boot TestContainers With Kafka](https://piotrminkowski.com/2019/10/09/part-1-testing-kafka-microservices-with-micronaut/)**
