# 1. 什么是 elasticsearch ?
Solr, ES 基于lucene开发的，全文检索服务器
lucene 是基于java开发的全文检索API

什么是全文检索？
将一段词语进行分词，并且将分出的单个词语统一的放到一个分词库中，在搜索时，根据关键字去分词库中检索，找到匹配的内容。 (倒排索引)

什么是倒排索引？
就是 不会直接到数据中去检索，而是先到分词库中去检索，因此叫倒排索引。


# 2. 使用方式 
安装、配置即可 > 因为ES是个服务器。

# 3.ES 和 Solr 对比
solr 实时性 比 es 差。 比如说：更新了一个文件,通过solr,不能实时查询到。
solr 在传统查询中 比 es 好。

solr 搭建集群，需要依赖zookeeper来管理
es 本身支持集群搭建，不需要第三方介入。

es 文档多
solr 针对国内文档不多

es对现在云计算和大数据支持特别好

# 4. ES安装
官方地址: https://www.elastic.co/cn/elasticsearch/

# 5. 安装好后
bin   # 可执行命令文件
eslasticsearch/config/elasticsearch.yml    #配置文件
lib   # 就是es使用到的jar包
module    # 就是一些扩展模块

启动需jdk, 要求jdk 1.8以上, 因为es是基于java开发的。 

# ElasticSearch 中文指南
https://www.elastic.co/guide/cn/elasticsearch/guide/current/foreword_id.html


## 安装
# 获取ES的镜像地址
进入该 http://hub.daocloud.io/  >  搜索 elasticsearch  >  社区镜像  >  版本
就可以得到关于ES的镜像地址

```
version: "3.1"
services: 
    elasticsearch: 
        image: daocloud.io/library/elasticsearch:6.5.4
        restart: always    
        container_name: elasticsearch
        port:
            - 9200:9200
            
    kibana:
        image: daocloud.io/library/kibana:6.5.4
            restart: always    
            container_name: kibana
            port:
                -5601:5601
            environment:
                - elasticsearch_url = http://192.168.199.109:9200
            depends_on:
                - elasticsearch
```
cd /opt
mkdir docker_es
cd docker_es
vi docker-compose.xml

## 安装分词器
cd ES/bin
./elasticsearch-plugin install http://tomcat01.qfjava.cn:91/elasticsearch-analysis-ik-6.5.4.zip
下载ik分词器的地址： 版本与es kibana一致。
http://tomcat01.qfjava.cn:91/elasticsearch-analysis-ik-6.5.4.zip



## ES的结构

ES会对索引进行分片， 还会对索引进行备份。

ES服务中，可以创建多个索引。 
每个索引默认分成5片存储。 
每个分片会存储至少一个备份分片。  
备份的分片必须放在不同的服务器中。


# 对应关系

索引--数据库
类型--表
文档document--行
属性field--列


## Restful
根据请求方式的不同，从而决定操作的业务是什么。

## 操作ES的Resful语法
GET请求
http://ip:port/index : 查询索引信息
http://ip:port/type/doc_id  : 查询文档信息

POST请求
http://ip:port/index/type/_search : 查询文档，可以在请求体中添加json字符串来代表查询条件
http://ip:port/index/type/doc_id/_update  :  修改文档，在请求体中指定json字符串代表修改的具体信息。

PUT请求
http://ip:port/index  :  创建一个索引， 需要在请求体中指定索引的信息
http://ip:port/index/type/_mappings  : 代表创建索引时，指定索引文档存储的属性的信息

DELETE请求:
http://ip:port/index : 删除跑路
http://ip:port/index/type/doc_id  : 删除指定的文档


## 实战代码  kibana

创建一个索引
PUT /person
{
    "settings":{
        "number_of_shards": 5,
        "number_of_replicas": 1
    }
}

查看索引信息
GET /person

删除索引
DELETE /person

## ES中的Field类型

String
    text: 一般被用于全文检索，将当前Field进行分词
    keyword: 当前Field不会被分词
    

#创建所以并制定结构
创建一个索引为 book 的，类型是小说 novel 的结构， 类型中又有哪些文档的属性
```
PUT /book
{
    "settings":{
        # 分片数
        "number_of_shards": 5,
        # 备份数
        "number_of_replicas": 1
    }
    "mappings":{
    
        # 类型Type
        "novel" :{
        
            # 文档存储的 Field
            "properties":{
            
                # Field属性名
                "name": {
                
                    # 类型
                    "type" : "text",
                    
                    # 指定分词器
                    "analyzer": "ik_max_word",
                    
                    # 指定当前Field可以被作为查询条件
                    "index" : true    
                    
                    # 表示当前的filed是否需要进行额外的存储
                    "store" : false    
                }
                
                "author":{
                    "type" : "keyword"
                }
                
                "on-sale":{
                    "type" : "date",
                    # 时间类型的格式化方式
                    "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
                }
                                
            }
        }
    }
}
```

# 文档操作
文档在ES服务中的唯一标识， _index 索引   _type 类型    _id 每个文档都是具有一个ID的
```
1. 新建文档
自动生成_id   
# 添加一个文档到novel中
POST /book/novel
{
    "name":"Forrest",
    "author": "西红柿",    
}

手动指定_id   请求方式 PUT
PUT /book/novel
{
    "name":"Forrest",
    "author": "麻花团队",  
}

2. 修改文档

覆盖式修改
PUT /book/novel
{
    "name":"Forrest",
    "author": "麻花团队",  
}

基于doc方式
POST /bood/novel/1/_update{
    "doc":{
        # 指定上需要修改的field和对应的值
        "name": "Mr S"
    }
}

3. 删除文档
根据id删除文档
DELETE /book/novel/1


```


## java 操作 ElasticSearch

```
1. Java 连接 ES
创建Maven工程 > 导入依赖（elasticsearch   elasticsearch的高级API  junit  lombok）

找依赖到  https://mvnrepository.com/


2. 创建测试类，连接ES

    public static RestHighLevelClient getClient(){
    
        // 创建HttpHost对象
        HttpHost httpHost = new HttpHost(hostname: "192.168.199.109", port: 9200)
        
        // 创建RestClientBuilder
        RestClientBuilder clientBuilder = RestClient.builder(httpHost)
        
        // 创建RestHighLevelClient
        RestHightLevelClient client = new RestHightLevelClient(clientBuilder)
        
        // 返回
        return client;
    
    }



    @Test
    public void testConnect(){
    
        RestHighLevelClient client = ESClient.getClient();
        System.out.println("OK")
    
    }
    
    
    @Test
    public void delete(){
    
        //1. 准备request对象
        DeleteIndexRequest request = new DeleteIndexRequest();
        request.indices(index);
        
        //2. 通过client对象执行
        AckonwledgedResponse delete = client.indices().delete(request, RequestOptions.DEFAULT)
        
        //3. 获取返回结果
        sysout(delete)
    }
    
    
    # 检查索引
    @Test
    public void exists(){
    
        //1. 准备request对象
        GetIndexRequest request = new GetIndexRequest();
        request.indices(index)
        
        //2. 通过client去操作
        boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);
        
        //3. 输出        
        sysout(exists)
    }
    
    String index = "person"
    String type = "man"
    @Test
    public void createIndex(){
    
        // 1. 准备关于索引的settings
        Settings.builder().put("number_of_shards", 3).put("number_of_replicas", 1);
        
        // 2. 准备关于索引的结构 mappings
        XContentBuilder mappings = JsonXContent.contentBuilder()
        .startObject()
            .startObject("properties")
                .startObject("name")
                    .field(name:"type", value:"text")
                .endObject()
                
                
            .endObject()
        .endObject()
        
        
        // 3. 将settings 和 mapping封装到一个 request对象
        CreateIndexRequest request = new CreateIndexRequest(index)
            .settings(settings)
            .mapping(type, mappings);
        
        // 4. 通过client对象去连接ES并执行创建索引
        CreateIndexResponse resp = client.indices().create(request, RequestOptions.DEFAULT);
        
        // 5. 输出
        sysout("resp" + resp.toString())
    }
    
```