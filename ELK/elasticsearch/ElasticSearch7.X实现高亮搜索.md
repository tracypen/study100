![](https://upload-images.jianshu.io/upload_images/8387919-98e040be42cf8ef3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>  高亮查询是搜索引擎中最基础的也是最重要的一个功能，站内搜索、电商等都可以通过高亮查询来提升用户体验。在`ElasticSearch`中高亮查询（highlight）是与`query`平级的查询
- 基础语法如下
```
GET /_index/_search 
{
 "query": {
    "match": {
      "address": "循化县"
    }
  },
  "highlight" : {
       "fields": {
          "_FIELD":{
           "type": "unified"
          }
      },
      "pre_tags": ["<font color='red'>"],
       "post_tags": ["</font>"]
  }
}
```

#### 示例
在索引库`check_station_index`中`address`字段被定义为text类型，也就是说可以分词
```
"address" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
```
- 高亮查询
```
GET /check_station_index/_search
{
  "query": {
    "match": {
      "address": "循化县"
    }
  }, 
  "highlight": {
    "fields": {
      "address":{
         "type": "unified"
      }
    },
    "pre_tags": ["<font color='red'>"],
    "post_tags": ["</font>"]
    
  }
}
```
- 查询结果
```
   {
        "_index" : "check_station_index",
        "_type" : "_doc",
        "_id" : "1279939719420968994",
        "_score" : 8.563546,
        "_source" : {
          "id" : "1279939719420968994",
          "waterDepartment" : "黄河干流",
          "riverName" : "黄河",
          "num" : "40100550",
          "name" : "循化(二)",
          "stationType" : "水文",
          "address" : "青海省循化县积石镇",
          "longitude" : "102°30′",
          "latitude" : "35°50′",
          "buildDate" : "1945.10",
          "waterNet" : "黄河流域"
        },
        "highlight" : {
          "address" : [
            "青海省<font color='red'>循</font><font color='red'>化</font><font color='red'>县</font>积石镇"
          ]
        }
      }
```
可以看到结果hits中`highlight`字段中返回了对应的高亮字段结果，并添加了指定的前缀和后缀，这里定义了红色字体样式，这样在前端展示的时候可以自动呈现为红色的

#### 通过`Java HighLevel Api`进行查询
- 1. 在`Service`中编写address高亮查询
```
@Override
    public List<Map<String, Object>> highlighted(String address) throws IOException {
        //定义索引库  
        SearchRequest searchRequest = new SearchRequest("check_station_index");
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        //定义query查询
        QueryBuilder queryBuilder = QueryBuilders.matchQuery("address", address);
        //定义高亮查询
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        //设置需要高亮的字段
        highlightBuilder.field("address")
                // 设置前缀、后缀
                .preTags("<font color='red'>")
                .postTags("</font>");
        searchSourceBuilder.query(queryBuilder);
        searchSourceBuilder.highlighter(highlightBuilder);
        searchRequest.source(searchSourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);

        List<Map<String, Object>> list = Lists.newArrayList();
        //遍历高亮结果
        for (SearchHit hit : searchResponse.getHits().getHits()) {
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            HighlightField nameHighlight = highlightFields.get("address");
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();

            if (nameHighlight != null) {
                Text[] fragments = nameHighlight.getFragments();
                String _address = "";
                for (Text text : fragments) {
                    _address += text;
                }
                sourceAsMap.put("address", _address);
                list.add(sourceAsMap);
            }
        }
        return list;
    }[图片上传中...(thumb.jpg-eeb740-1595602689859-0)]

```
- 编写测试类
```
/**
 * @author haopeng
 * @date 2020-07-24 22:09
 */
@SpringBootTest
@RunWith(SpringRunner.class)
public class HighLightTest {

    @Autowired
    private WeatherEsService weatherEsService;

     @Test
     public void highlight() throws IOException {
         List<Map<String, Object>> highlighted = weatherEsService.highlighted("循化县");
         highlighted.forEach(System.out::println);
     }

}
```
- 运行结果
```
{address=青海省<font color='red'>循</font><font color='red'>化</font><font color='red'>县</font>积石镇, stationType=水文, num=40100550, latitude=35°50′, name=循化(二), waterNet=黄河流域, id=1279939719420968994, buildDate=1945.10, waterDepartment=黄河干流, riverName=黄河, longitude=102°30′}
{address=青海省<font color='red'>循</font><font color='red'>化</font><font color='red'>县</font>梁什滩, stationType=水文, num=40203200, latitude=35°48′, name=文都(三), waterNet=黄河流域, id=1279939720733786132, buildDate=1960.04, waterDepartment=黄河上游区上段, riverName=街子河, longitude=102°24′}
{address=青海省<font color='red'>循</font><font color='red'>化</font><font color='red'>县</font>道帏乡古雷村, stationType=雨量, num=40228300, latitude=35°39′, name=道帏, waterNet=黄河流域, id=1279939719420968981, buildDate=1985.01, waterDepartment=黄河上游区上段, riverName=清水, longitude=102°39′}
```

![](https://upload-images.jianshu.io/upload_images/8387919-8b5fcfd8f1c82841.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/8387919-6b8af200e979530a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)