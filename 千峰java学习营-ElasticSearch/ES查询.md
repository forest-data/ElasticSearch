
# term&terms查询

term查询 : 代表完全匹配，搜索之前不会对关键字进行分词，对你的关键字去文档分词库中去匹配内容。

POST /index/type
{
    "from": 0,   # limit ?
    "size": 5,   # limit x.?
    "query":{
        "term": {
            "province": {
                "value": "beijing"
            }
        }
    }
}

terms 和 term 的查询机制是一样， 都不会将指定的查询关键字进行分词， 直接去分词库中匹配， 找到相应文档内容。
terms 是在针对一个字段包含多个值的时候使用。

term: where provice = 北京；
terms: where provice = beijing or provice = ? or provice = ?

POST /index/type
{
    "from": 0,   # limit ?
    "size": 5,   # limit x.?
    "query":{
        "terms": {
            "province": [
                "beijing", "shanxi" ...
            ]
        }
    }
}

不要认为 term 和 terms 只针对keyword, 普通的text也可以，不过注意 term是会分词，将分词去分词库中查询

# match 查询
match 查询属于高层查询， 他会根据你查询的字段类型不一样，采用不同的查询方式。
    查询的是日期或者是数值的话， 他会将你基于的字符串查询内容转换为日期或者数值对待。
    如果查询的内容是一个不能被分词的内容（keyword), match查询不会对你指定查询关键字进行分词。
    如果查询的内容是一个可以被分词的内容(text), match会将你指定的查询内容根据一定的方式分词，去分词库中匹配指定的内容。
    
match查询， 实际底层就是多个term查询，将多个查询的结果给你封装到了一起。

POST /index/type
{
    "query":{
        "match": {
            "smsContent" : 中国
        }
    }
}

# 布尔 match 查询 
基于一个Field匹配的内容，采用 and 或者 or 的方式
POST /index/type
{
    "query":{
        "match": {
            "smsContent" : {
                "query" : "中国  健康"，
                "operator" : "and"    # 内容即包含中国也包含健康
            }
        }
    }
}

# multi_match 查询
match针对一个field做检索， multi_match针对多个field进行检索，多个field对应一个text文本.
POST /index/type
{
    "query":{
        "multi_match": { 
             "query" : "中国  健康"，   # 指定text
             "fields": ["province", "smsContent"]     # 指定field       
        }
    }
}


# match_all 查询
查询全部内容，不指定任何查询条件


POST /index/type
{
    "query":{
        "match_all": {}
    }
}

# id查询
类似 where id = ? 
GET  /index/type/1

# ids查询
根据多个id查询，类似mysql中的 where id in (id1, id2, id3)

POST /index/type
{
    "query":{
        "ids": {
            "values": [1,2,3]
        }
    }
}


# prefix 查询  
前缀查询， 可以通过一个关键字去指定一个Field大的前缀，从而查询到指定的文档。

POST /index/type/_search
{
    "query":{
        "prefix": {
            corpName: {value : 途虎}
        }
    }
}


# fuzzy查询
模糊查询，我们输入字符的大概，ES就可以取根据输入的内容大概去匹配一下结果

POST /index/type/_search
{
    "query":{
        "fuzzy": {
            "corpName": {         
                "value" : 途虎,
                "prefix_length": 2   # 指定前面几个字符是不允许出现错误的。      
            }
        }
    }
}


# wildcard查询
通配查询，和mysql中的like是一个套路，可以在查询时，在字符串中指定 通配符* 和 占位符 ？


POST /index/type/_search
{
    "query":{
        "wildcard": {
            "corpName": {         
                "value" : 中国？？    # 查中国移动     可以使用 * 和 ？ 指定通配符 和 占位符 
            }
        }
    }
}


# range 查询
范围查询，只针对数值类型，对某一个字段Field进行大于或者小于的范围指定。

POST /index/type/_search
{
    "query":{
        "range": {
            "fee": {         
                "gte": 10,
                "lte": 20
                # 可以使用gt: >   gte: >=    lt: <    lte: <=
            }
        }
    }
}


# regexp 查询
正则查询， 通过你编写的正则表达式去匹配内容。
    prefix, wildcard 和 regexp 查询效率相对比较低，要求效率比较高时， 避免去使用。


POST /index/type/_search
{
    "query":{
        "regexp": {
            "mobile": "180[0-9]{8}"    # 编写正则
        }
    }
}



## 深分页Scroll
ES对 from + size 是有限的， from 和 size 二者之和 不能超过1W
原理：
    ES查询数据的方式 
    第一步 现将用户指定的关键进行分词。
    第二步 将词汇去分词库中进行检索，得到多个文档的id。
    第三步 去各个分片中去拉取指定的数据。    耗时较长
    第四步 将数据根据score进行排序。    耗时较长
    第五步 根据from的值，将查询到的数据舍弃一部分。 
    第六步 返回结果。

Scroll在ES查询数据方式：
    第一步 现将用户指定的关键进行分词
    第二步 将词汇去分词库中进行检索，得到多个文档的id
    第三步 将文档的id存放在一个ES的上下文中
    第四步 根据你指定size的个数去ES中检索指定的数据，拿完数据的文档id, 会从上下文中移除。
    第五步 如果需要下一页数据，直接去ES的上下文中，找后续内容
    第六步 循环 第4步 第5步。
    (Scroll查询方式， 不适合做实时的查询)

#执行scroll查询， 返回第一页数据， 并且将文档id信息存放在ES上下文中，并制定生存时间。
POST /index/type/_search?scroll=1m   # 表示现在要做scroll分页查询，1m表示查到的文档id 要存到ES上下文中1分钟。
```json
{
    "query":{
        "match_all" : {}
    },
    "size" : 2,
    "sort" : [{     # 排序
        "fee":{
            "order" : "desc"
        }
    }]
}
```

#根据scroll查询第二页数据
POST /_search/scroll
```json
{
    "scroll_id" : <根据第一步得到的scorll_id去指定>
    "scroll" : "1m"  <scroll信息的生存时间>
}
```
#删除scroll在ES上下文中的数据
DELETE /_search/scroll/~

# delete-by-query
根据term, match 等查询方式去删除大量的文档
如果你需要删除的内容，是index下的大部分数据， 推荐创建一个全新的index, 将保留的文档内容，添加到新的index

POST /index/type/_delete_by_query
```json
{
    "query":{
        "range": {
            "fee": {
                "lt" : 4
            }
        }
    }
}
```

## 复合查询

# bool查询
复合过滤器， 将你的多个查询条件，以一定的逻辑组合在一起， and or not 
must : 所有条件，用must组合在一起， 表示And的意思
must_not : 将must_not的条件，全部都不能匹配，标识Not的意思
should: 所有的条件， 用should组合在一起，表示or的意思

POST /index/type/_search
```json
{
    "query":{
        "bool": {
            "should": [
            {
                "term" : {
                    "province": {
                        "value" : "北京"
                    }
                }
            },
            {
                "term":{
                    "province": {
                        "value" : "武汉"
                    }
                }
            }
            ],
            
            "must_not": [
            {
                "term": {
                    "operatorId":{
                        "value": 2
                    }
                }
            } 
            ],
            
            "must": [
            {
                "match" : {
                    "smsContent" : "平安"
                }
            },
            {
                "match" : {
                    "smsContent" : "china"
                }
            }
            ]
        }
    }
}
```

# boosting查询
boosting查询可以帮助我们去影响查询后的score

    positive : 只有匹配上positive的查询的内容， 才会被放到返回的结果集中
    negatie : 如果匹配上和positive并且也匹配上了negative,就可以降低这样的文档score
    negative_boost : 指定系数， 必须小于1.0
    
    关于查询时， 分数是如何计算的
    搜索的关键字在文档中出现的频次越高， 分数就越高
    指定的文段内容越短，分数就越高
    我们在探索时，指定的关键字也会被分词，这个被分词的内容，被分词库匹配的个数越多，分数越高。

POST /index/type/_search
```json


{
    "query":{
        "boosting": {
            "positive": {
                "match" : {
                    "smsContent" : "收货安装"
                }
            },
            
            "negative" : {
                "match": {
                    "smsContent" : "王五"
                }
            },
            
            "negative_boost" : 0.5 
            
        }
    }
}
```

# filter查询
query，根据你的查询条件，去计算文档的匹配度得到一个分数， 并且根据分数进行排序， 不会做缓存的。
filter，根据你的查询条件去查询文档，不去计算分数，而且filter会对经常被过滤的数据进行缓存。
POST /index/type/_search
```json

{
    "query":{
        "bool": {
            "filter": {
                "term" : {
                    "corpName" : "盒马鲜生"
                }
            },
            
            "range" : {
                "fee" : {
                    "lte" : 4
                }
            }            
        }
    }
}
```


# 高亮查询
高亮查询，就是将用户输入的关键字，以一定的特殊样式展示给用户，让用户知道为什么这个结果被检索出来。
高亮展示的数据，本身就是文档中的一个Field, 单独将Field以highlight的形式返回给你。

ES提供了一个highlight属性， 和query同级别的
    fragment_size : 指定高亮数据 展示多少个字符回来。
    pre_tags : 指定前缀标签 <font color="red">
    post_tags : 指定后缀标签 </font>
    fields: 指定哪个field以高亮形式返回
    
POST /index/type/_search    
```json
{
    "query":{
        "match": {
                "smsContent" : "盒马"
        }
    }
    
     "highlight" : {
         "fields" : {
             "smsContent" : {}
         },
         
         "pre_tags" : "<font color = 'red'>",
         "post_tags" : "</font>",
         "fragment_size": 10        
     }
     
}
```   

# 聚合查询
ES的聚合查询 和 Mysql的聚合查询类型，ES的聚合查询相比Mysql要强大的多，ES提供的统计数据的方式多种多样。
POST /index/type/_search
```json
{
    "aggs":{
        
        "agg名字" : {
            "agg_type": {
                "属性" : "值"
            }
        }
        
    }
    
}
```

# 去重计数查询

去重计数， 即Cardinality, 第一步先将返回的文档中的一个指定的field进行去重。统计一共有多少条。
POST /index/type/_search
```json
{
    "aggs":{
        
        "agg" : {
            "cardinality": {
                "field" : "province"
            }
        }
        
    }
    
}
```

# 范围统计
统计一定范围内出现的文档个数， 比如，针对某一个Field的值在 0-100， 100-200， 200-300之间文档出现的个数分别是多少。

范围统计可以针对普通的数值，针对时间类型，针对ip类型都可以做相应的统计
range, date_range, ip_range



POST /index/type/_search
```json
{
    "aggs":{
        "agg" : {
            "range": [
              {
                "to" : 5
              },
              { 
                "from" : 5,   # from 是包含   5之后
                "to" : 10     # to是不包含    10之前
              },
              {
                "from" : 10 
              }
            ]
        }        
    }    
}
```

时间方式范围统计
POST /index/type/_search
```json
{
    "aggs":{
        "agg" : {
            "date_range": {          
                "field" : "createDate",
                "format": "yyyy",
                
                "ranges": [
                  {     
                    "to" : 2000  
                  },
                  {
                    "from" : 2000
                  }
                ]
               
            }
        }        
    }    
}
```

ip方式 范围统计
POST /index/type/_search
```json
{
    "aggs":{
        "agg" : {
            "ip_range": {          
                "field" : "ipAddr", 
                
                "ranges": [
                  {     
                    "to" : "10.126.8.2"
                  },
                  {
                    "from" : "10.126.8.1"
                  }
                ]
               
            }
        }        
    }    
}
```


# 统计聚合查询
他可以帮你查询指定Field的最大值， 最小值，平均值，平方和

POST /index/type/_search
```json
{
    "aggs":{
        "agg" : {
            "extended_stats": {          
                "field" : "fee" 
            }
        }        
    }    
}
```

# 地图经纬度搜索
ES中提供了一个数据类型 geo_point, 这个类型就是用来存储经纬度的。
创建一个袋geo_point类型的索引，并添加测试数据

创建一个索引，指定一个name, location

PUT /map
```json
{
    "settings": {
        "number_of_shards" : 5,
        "number_of_replicas" : 1
    },
    "mappings": {    
        "map": {
            "properties": {
                "name": {
                    "type": "text"
                },
                
                "location": {
                    "type" : "geo_point"              
                }
            }
          
        }    
    }    
}
```

添加测试数据

PUT /map/map/1
```json
{
    "name" : "天安门",
    "location" : {
        "lon" : 116,
        "lat" : 39
    }
}
```


ES的地图检索方式
geo_distance : 直线距离检索方式   圆
geo_bounding_box: 以两个点确定一个矩形，获取在矩形内的全部数据。
geo_polygon: 以多个点，确定一个多边形，获取多边形内的全部数据

POST /map/map/_search
```json
{
    "query" : {
        "geo_distance" : {
            "location" : {    # 确定一个点
                "lon" : 116,
                "lat" : 39
            }
            "distance" : 3000,    # 确定距离 
            "distance_type" : "arc"    # 指定形状为圆形
        }
    }
}

```
