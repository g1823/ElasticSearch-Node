## 基础操作

​	索引相当于数据库，创建索引就是创建一些数据库。

​	然后给对应的索引设置映射，映射就相当于是给索引设置一些规则(文档字段的存储方式、查询方式和分析方式)，就像是设置数据库的字符集、是否开启慢sql监控等。

​	之后给索引上传文档，也就是我们需要解析的文档，上传文档后，文档会根据索引设置的各种规则(分词规则、映射信息)生成倒排索引，便于查询。

​	也就是说，可以把 **Elasticsearch 索引** 看作是一个 **数据库**，每个索引有自己的 **映射（Mapping）**、**分词规则（Analyzer）**、**设置（Settings）** 等信息。而上传文档到指定索引时，就相当于按照该索引的规则来存储数据，生成倒排索引，并在查询时依赖于这些规则来执行搜索。

**创建索引或上传文档时指定的索引 和 上传文档后生成的倒排索引的区别：**

1. **上传文档到指定的索引**：

   - 这个“指定的索引”是你在上传文档时提供的索引名称，它决定了文档在哪个逻辑容器中存储。索引本质上是 Elasticsearch 存储文档和元数据（如倒排索引）的容器。

   例如，如果你上传文档到 `products` 索引，那么 Elasticsearch 会根据该索引的映射（Mapping）和设置（Settings）来存储文档，并在内部生成倒排索引。

2. **倒排索引**：

   - 对于每个被上传的文档，Elasticsearch 会对其内容（尤其是文本字段）进行解析，生成倒排索引。倒排索引的生成是基于文档中各个字段的内容（如文本分词后形成的词项）。这些倒排索引是存储在指定索引内的。

即：

- **上传文档指定索引**：指定文档应该属于哪个索引，这个索引为文档提供了映射规则、分析器等配置。
- **生成倒排索引**：文档上传后，文档字段会根据映射规则和分析器生成倒排索引，倒排索引是用来加速搜索的。

## Java-Api操作

​	每个api请求数据以及返回数据其实与Http操作类似，只是将相关数据封装成了对象，因此对于请求体和响应体可以参考[2-ES入门-2-基础操作-Http操作](./2-ES入门-2-基础操作-Http操作.md)

### 1、引入依赖

​	这里以7.8.0版本为例，进行演示。其中各个依赖介绍：

- `elasticsearch`： `Elasticsearch` 依赖
- `elasticsearch-rest-high-level-client`：`Elasticsearch Rest`高级客户端
- `log4j`：日志框架，`Elasticsearch`内部需要使用到日志框架
- `jackson-databind`：xml解析

#### 1.1 maven版本

`pom.xml`文件：

~~~ xml
<dependencies>
    <dependency>
        <groupId>org.elasticsearch</groupId>
        <artifactId>elasticsearch</artifactId>
        <version>7.8.0</version>
    </dependency>
    <!-- elasticsearch 的客户端 -->
    <dependency>
        <groupId>org.elasticsearch.client</groupId>
        <artifactId>elasticsearch-rest-high-level-client</artifactId>
        <version>7.8.0</version>
    </dependency>
    <!-- elasticsearch 依赖 2.x 的 log4j -->
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
        <version>2.8.2</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.8.2</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.9.9</version>
    </dependency>
    <!-- junit 单元测试 -->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
</dependencies>
~~~

#### 1.2 gradle版本

~~~ groovy
dependencies {
    // Elasticsearch 依赖
    implementation 'org.elasticsearch:elasticsearch:7.8.0'

    // Elasticsearch 的 REST 高级客户端
    implementation 'org.elasticsearch.client:elasticsearch-rest-high-level-client:7.8.0'

  	// log4j 依赖
    implementation 'org.apache.logging.log4j:log4j-api:2.15.0'
    implementation 'org.apache.logging.log4j:log4j-core:2.15.0'
  
    // Jackson 依赖
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.9.9'

    // JUnit 单元测试
    testImplementation 'junit:junit:4.12'
}
~~~

### 2、创建客户端

​	通过静态代码块关闭ES客户端，避免资源浪费。

​	RestHighLevelClient为7.x版本使用，8.x已经不再使用。

~~~ java
public class ESClientUtil {
    private static volatile RestHighLevelClient client = null;
    private ESClientUtil() {
    }
    static {
        // 注册一个 JVM 关闭时的钩子
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            if (client != null) {
                System.out.println("JVM 关闭，释放资源...");
                try {
                    client.close();
                } catch (IOException e) {
                    System.out.println("关闭失败：" + e.getMessage());
                }
            }
        }));
    }
    public static RestHighLevelClient getClient() {
        if (client == null) {
            synchronized (ESClient.class) {
                if (client == null) {
                    client = new RestHighLevelClient(
                            RestClient.builder(new HttpHost("localhost", 9200, "http"))
                                    .setRequestConfigCallback(requestConfigBuilder ->
                                            requestConfigBuilder
                                                    .setConnectTimeout(5000)  // 连接超时（默认1秒）
                                                    .setSocketTimeout(60000)  // 读取超时（默认30秒）
                                    )
                                    .setHttpClientConfigCallback(httpClientBuilder ->
                                            httpClientBuilder.setMaxConnTotal(100)  // 最大连接数
                                                    .setMaxConnPerRoute(10)  // 每个主机的最大连接数
                                    )
                    );
                }
            }
        }
        return client;
    }
}
~~~

### 3、索引操作

#### 3.1 创建索引

~~~ java
		/**
     * 创建索引
     */
    public void createIndex(RestHighLevelClient client) throws IOException {
        // 注意，索引名称不能存在大写单词，可以用"_"来间隔，否则报错：
        CreateIndexRequest request = new CreateIndexRequest("index_name");
        // 发送请求，获取相应
        CreateIndexResponse response = client.indices().create(request, RequestOptions.DEFAULT);
        // 输出响应是否被确认
        System.out.println(response.isAcknowledged());
    }
~~~

#### 3.2 查看索引

~~~ java
    /**
     * 获取索引信息
     */
    public void getIndex(RestHighLevelClient client) throws IOException {
        GetIndexRequest getIndexRequest = new GetIndexRequest("index_name");
        GetIndexResponse getIndexResponse = client.indices().get(getIndexRequest, RequestOptions.DEFAULT);
        // getAliases() 方法返回与该索引关联的所有别名
        System.out.println(getIndexResponse.getAliases());
        // getMappings() 方法返回该索引的字段映射（如字段类型、分析器等）
        System.out.println(getIndexResponse.getMappings());
        // getSettings() 方法返回索引的配置参数（如分片数、副本数等）
        System.out.println(getIndexResponse.getSettings());
    }
~~~

#### 3.3 删除索引

~~~ java
    /**
     * 删除索引
     */
    public void deleteIndex(RestHighLevelClient client) throws IOException {
        // 删除索引 - 请求对象
        DeleteIndexRequest request = new DeleteIndexRequest("index_name");
        // 发送请求，获取响应
        AcknowledgedResponse response = client.indices().delete(request,
                RequestOptions.DEFAULT);
        // 操作结果
        System.out.println("操作结果 ： " + response.isAcknowledged());
    }
~~~

### 4、文档操作

#### 4.1 创建文档

​	User是对应实体类，通过ObjectMapper转为Json字符串。

~~~ java
/**
     * 创建文档
     * @param client 客户端
     */
    public void createDocument(RestHighLevelClient client) throws IOException {
        // 新增文档 - 请求对象
        IndexRequest request = new IndexRequest();
        // 设置索引及唯一性标识
        request.index("user").id("001");
        // 创建数据对象
        User user = new User();
        user.setName("zhangsan");
        user.setAge(30);
        user.setSex("男");
        ObjectMapper objectMapper = new ObjectMapper();
        String productJson = objectMapper.writeValueAsString(user);
        // 添加文档数据，数据格式为 JSON 格式
        request.source(productJson, XContentType.JSON);
        // 客户端发送请求，获取响应对象
        IndexResponse response = client.index(request, RequestOptions.DEFAULT);
        // 打印结果信息
        System.out.println("_index:" + response.getIndex());
        System.out.println("_id:" + response.getId());
        System.out.println("_result:" + response.getResult());
    }
~~~

批量新增：

~~~ java
		/**
     * 批量创建文档
     */
    public void batchCreateDocument(RestHighLevelClient client) throws IOException {
        //创建批量新增请求对象
        BulkRequest request = new BulkRequest();
        request.add(new IndexRequest().index("user").id("1001").source(XContentType.JSON, "name", "zhangsan"));
        request.add(new IndexRequest().index("user").id("1002").source(XContentType.JSON, "name", "lisi"));
        request.add(new IndexRequest().index("user").id("1003").source(XContentType.JSON, "name", "wangwu"));
        //客户端发送请求，获取响应对象
        BulkResponse responses = client.bulk(request, RequestOptions.DEFAULT);
        //打印结果信息
        System.out.println("took:" + responses.getTook());
        System.out.println("items:" + responses.getItems());
    }
~~~



#### 4.2 修改文档

~~~ java
		/**
     * 更新文档
     */
    public void updateDocument(RestHighLevelClient client) throws IOException {
        UpdateRequest request = new UpdateRequest();
        // 配置修改参数
        request.index("user").id("001");
        // 设置请求体，对数据进行修改
        request.doc(XContentType.JSON, "sex", "女");
        // 客户端发送请求，获取响应对象
        UpdateResponse response = client.update(request, RequestOptions.DEFAULT);
        System.out.println("_index:" + response.getIndex());
        System.out.println("_id:" + response.getId());
        System.out.println("_result:" + response.getResult());
    }
~~~

#### 4.3 查看文档

~~~ java
		/**
     * 查询文档
     */
    public void getDocument(RestHighLevelClient client) throws IOException {
        //1.创建请求对象
        GetRequest request = new GetRequest().index("user").id("001");
        //2.客户端发送请求，获取响应对象
        GetResponse response = client.get(request, RequestOptions.DEFAULT);
        //3.打印结果信息
        System.out.println("_index:" + response.getIndex());
        System.out.println("_type:" + response.getType());
        System.out.println("_id:" + response.getId());
        System.out.println("source:" + response.getSourceAsString());
    }
~~~

#### 4.4 删除文档

~~~ java
    /**
     * 删除文档
     */
    public void deleteDocument(RestHighLevelClient client) throws IOException {
        //创建请求对象
        DeleteRequest request = new DeleteRequest().index("user").id("001");
        //客户端发送请求，获取响应对象
        DeleteResponse response = client.delete(request, RequestOptions.DEFAULT);
        //打印信息
        System.out.println(response.toString());
    }
~~~

批量删除：

~~~ java
		/**
     * 批量删除文档
     */
    public void batchDeleteDocument(RestHighLevelClient client) throws IOException {
        //创建批量删除请求对象
        BulkRequest request = new BulkRequest();
        request.add(new DeleteRequest().index("user").id("1001"));
        request.add(new DeleteRequest().index("user").id("1002"));
        request.add(new DeleteRequest().index("user").id("1003"));
        //客户端发送请求，获取响应对象
        BulkResponse responses = client.bulk(request, RequestOptions.DEFAULT);
        //打印结果信息
        System.out.println("took:" + responses.getTook());
        System.out.println("items:" + responses.getItems());
    }
~~~

### 5、查询操作

#### 5.1 各种类型查询

​	各种类型查询都是通过对SearchSourceBuilder对象，设置不同的query类型进行查询，最终得到SearchResponse对象。

~~~ java
    public void query(RestHighLevelClient client) throws IOException {
        // 创建搜索请求对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        // 构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();

        // 1、查询所有数据
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        request.source(sourceBuilder);
        SearchResponse queryAllResponse = client.search(request, RequestOptions.DEFAULT);

        // 2、精确查询
        sourceBuilder.query(QueryBuilders.termQuery("name", "zhangsan"));
        request.source(sourceBuilder);
        SearchResponse termQueryResponse = client.search(request, RequestOptions.DEFAULT);

        // 3、分页查询
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        // 当前页起始索引(第一条数据的顺序号)
        sourceBuilder.from(0);
        // 每页显示多少条 size
        sourceBuilder.size(2);
        request.source(sourceBuilder);
        SearchResponse pageQueryResponse = client.search(request, RequestOptions.DEFAULT);

        // 4、排序
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        sourceBuilder.sort("name", SortOrder.ASC);
        request.source(sourceBuilder);
        SearchResponse sortQueryResponse = client.search(request, RequestOptions.DEFAULT);

        // 5、过滤字段
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        //查询字段过滤
        String[] excludes = {};
        String[] includes = {"name", "age"};
        sourceBuilder.fetchSource(includes, excludes);
        request.source(sourceBuilder);
        SearchResponse filterQueryResponse = client.search(request, RequestOptions.DEFAULT);

        // 6、Bool查询。注意，这里使用了新的查询创建器BoolQueryBuilder
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        // 必须包含
        boolQueryBuilder.must(QueryBuilders.matchQuery("age", "30"));
        // 一定不含
        boolQueryBuilder.mustNot(QueryBuilders.matchQuery("name", "zhangsan"));
        // 可能包含
        boolQueryBuilder.should(QueryBuilders.matchQuery("sex", "男"));
        sourceBuilder.query(boolQueryBuilder);
        request.source(sourceBuilder);
        SearchResponse boolQueryResponse = client.search(request, RequestOptions.DEFAULT);

        // 7、范围查询
        RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("age");
        // 大于等于
        rangeQuery.gte("30");
        // 小于等于
        rangeQuery.lte("40");
        sourceBuilder.query(rangeQuery);
        request.source(sourceBuilder);
        SearchResponse rangeQueryResponse = client.search(request, RequestOptions.DEFAULT);

        // 8、模糊查询
        sourceBuilder.query(QueryBuilders.fuzzyQuery("name","zhangsan").fuzziness(Fuzziness.ONE));
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);

        // 查询匹配
        SearchHits hits = queryAllResponse.getHits();

        System.out.println("took:" + response.getTook());
        System.out.println("timeout:" + response.isTimedOut());
        System.out.println("total:" + hits.getTotalHits());
        System.out.println("MaxScore:" + hits.getMaxScore());
        System.out.println("hits========>>");
        for (SearchHit hit : hits) {
            //输出每条查询的结果信息
            System.out.println(hit.getSourceAsString());
        }
        System.out.println("<<========");
    }
~~~

#### 5.2 高亮查询

~~~ java
public void highLightQuery(RestHighLevelClient client) throws IOException {
        // 高亮查询
        SearchRequest request = new SearchRequest().indices("user");
        //2.创建查询请求体构建器
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //构建查询方式：高亮查询
        TermsQueryBuilder termsQueryBuilder = QueryBuilders.termsQuery("name","zhangsan");
        //设置查询方式
        sourceBuilder.query(termsQueryBuilder);
        //构建高亮字段
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.preTags("<font color='red'>");//设置标签前缀
        highlightBuilder.postTags("</font>");//设置标签后缀
        highlightBuilder.field("name");//设置高亮字段
        //设置高亮构建对象
        sourceBuilder.highlighter(highlightBuilder);
        //设置请求体
        request.source(sourceBuilder);
        //3.客户端发送请求，获取响应对象
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //4.打印响应结果
        SearchHits hits = response.getHits();
        System.out.println("took::"+response.getTook());
        System.out.println("time_out::"+response.isTimedOut());
        System.out.println("total::"+hits.getTotalHits());
        System.out.println("max_score::"+hits.getMaxScore());
        System.out.println("hits::::>>");
        for (SearchHit hit : hits) {
            String sourceAsString = hit.getSourceAsString();
            System.out.println(sourceAsString);
            //打印高亮结果
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            System.out.println(highlightFields);
        }
        System.out.println("<<::::");
    }
~~~

#### 5.3 聚合查询

~~~ java
public void aggsQuery(RestHighLevelClient client) throws IOException {
        // 设置请求索引
        SearchRequest request = new SearchRequest().indices("user");
        // 设置请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();

        // 1、最大值
        sourceBuilder.aggregation(AggregationBuilders.max("maxAge").field("age"));
        //设置请求体
        request.source(sourceBuilder);
        //客户端发送请求，获取响应对象
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);

        // 2、分组
        sourceBuilder.aggregation(AggregationBuilders.terms("age_groupby").field("age"));
        //设置请求体
        request.source(sourceBuilder);
        //客户端发送请求，获取响应对象
        SearchResponse response2 = client.search(request, RequestOptions.DEFAULT);
        
        //4.打印响应结果
        SearchHits hits = response.getHits();
        System.out.println(response);
    }
~~~



















