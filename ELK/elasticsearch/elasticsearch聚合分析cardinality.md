> cardinality 即去重计算，类似sql中 count（distinct），先去重再求和，计算指定field值的种类数。



```json
//聚合查询--基数查询 相当于查询有多种站点
//GET /sjy_index/_search
{
  "size": 0,
  "aggs": {
    "agg_years": {
      "cardinality": {
        "field": "name.keyword"
      }
    }
  }
}
```

