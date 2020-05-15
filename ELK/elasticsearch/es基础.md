>  在es7中已经移除了type的概念了

##### 自定义mapping

- index 控制当前字段是否被索引 设置为false后，不可搜索

```json
{
"mappings": {
	"doc":{
        "properties":{
            "uname": {
                type: "text"
                "index": false
            	}
        	}
        }
	}
}
```

- index_options 用于控制倒排索引的内容 有一下四种配置

```properties
- docs 只记录doc id
- freqs 记录doc id 和 term frequencies
- positions 记录doc id 和 term frequencies和term position
- offsets  记录doc id 和 term frequencies和term position 和character offsets

text类型默认为positions，其他默认为docs
记录的内容越多，占用内存越大
```

- null_value 

当字段为空值时的处理策略，默认为null时，es会忽略此字段，可以通过该项配置:

`"null_value": "NULL"`



##### es补充字段类型

- 专用类型
  - 记录IP地址 ip
  - 实现自动补全 completion
  - 记录分次数 token_count
  - 记录字符串hash值 numbur3
  - join
- 多字段类型muti-fields

允许对同一字段进行不同的配置，比如分词，常见的例子如对人名实现拼音搜索

只需在人名字段中新增一个子字段 `pinyin`

~~~json
{
    "propertise":{
        "username"{
        	"type" "text",
        	"fileds": {
        	"pinyin":{
        "type":"text",
        "analyzer":"pinyin"
    }
    }
    }
    }
}
~~~

搜索时可以使用`username.pinyin`进行检索



##### Dynamic Mapping

es是通过JSON文档的字段类型自动识别字段的额类型的

| JSON类型 | es类型                                                       |
| -------- | ------------------------------------------------------------ |
| null     | 忽略                                                         |
| boolean  | boolean                                                      |
| 浮点     | float                                                        |
| 整数     | long                                                         |
| object   | object                                                       |
| array    | 由第一个非null值得字段决定                                   |
| String   | 1.可以匹配日期 2.匹配float、long（需要额外设置）3.默认设置为**text**并附带**keyword**的子字段 |

##### 自定义Mapping的建议

1. 写入一条文档记录到index中，获取es自动生成的mapping
2. 修改上一步得到的mapping，自定义相关配置
3. 使用修改后的mapping创建实际的索引

##### 索引模板

索引模板API的endpoint为`_template` 他可以将mapping定义为一个模板，为属性相似的索引提供统一的配置

~~~json
PUT /_template/my_inde_template
{
  "index_patterns":["stu_*","user_*"],
	"order":"0",
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword"
      },
      "age": {
        "type": "long"
      }
    }
  }
}
~~~

- index_patterns 匹配索引名称

- order配置顺序

继续执行`put student_index` 之后通过`get student_index`查看到结果如下

~~~json
{
  "user_product" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "age" : {
          "type" : "long"
        },
        "name" : {
          "type" : "keyword"
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1589252047390",
        "number_of_shards" : "3",
        "number_of_replicas" : "0",
        "uuid" : "NB40ZSKGS1Kc8ehvCZJHUA",
        "version" : {
          "created" : "7010199"
        },
        "provided_name" : "user_product"
      }
    }
  }
}
~~~

##### searchAPI

>  es中search的endpoint为**_search**   例如：`GET /student/_search` 

ES中的Search分为两种 **URI Search**和**DSL**

###### URI Search

> 通过url参数控制查询，有部分局限性（先不学习了）

###### Query DSL

> 基于JSON定义的查询语言，主要包含以下两种查询：

1. **字段查询**

如term、match、range等，只针对一个字段进行查询

2. **复合查询**

如bool查询，包含一个或多个字段查询或复合查询的语句



- 1.字段查询

term查询（不评分）

~~~json
POST /person/_search
{
  "profile": "true", 
  "query": {
    "match_phrase": {
     "remark":{
       "query":"java python vue"
     }
    }
  }
}
~~~



match查询

~~~json
GET /person/_search
{
  "profile": "true", 
  "query": {
    "match": {
     "name":{
       "query":"Java python",
        "operator": "or/and",
       	"minimum_should_match": 2
     }
    }
  }
}
~~~

**说明：** 可以通过`"profile": "true"` 开启查询过程debug，可以看到具体的查询流程，方便进行调试

​			通过`"operator": "or/and",`指定单词间的匹配关系

​			通过`"minimum_should_match": 2` 控制单词最小匹配度

###### Match Query

![](https://upload-images.jianshu.io/upload_images/8387919-e4b0b0fd12fa972d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

-  相关性评分

> 计算文档与查询词语的相关度（符合程度）“relevance”,执行查询语句后，首先对查询的参数进行分词，然后通过倒排索引，找到对应的文档，并最终对文档进行评分，评分后按照分数的降序排列返回给客户端

评分的几个重要概念：

- Term Frequency(TF)词频 单词在文档中出现的次数，次数越高，相关度越高
- Document Frequency(DF) 文档频率 单词出现的文档的次数
- Inverse Document Frequency (IDF) 与文档频率相反，1/DF 可以理解为单词出现的文档数越少那么相关度就越高
- Field-length Norm 文档越短那么相关性就越高

相关性评分的两种算法：

- TF-IDF模型算法
- BM25模型算法（ES5.0以后默认采用）

> 注意：es的算分是shard独立的，就是说shard的分数互相独立

###### Match Phrase Query

> 对字段检索，但是有顺序要求

- 2.复合查询

布尔查询有一个或多个布尔子句组成

| filter   | 只进行过滤，不评分 |
| -------- | ------------------ |
| must     | 必须包含、影响评分 |
| must_not | 必须不包含         |
| should   | 可以包含、影响评分 |

