在es的home目录下

~~~properties
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.1.1/elasticsearch-analysis-ik-7.1.1.zip

~~~



重启es

~~~
POST http://47.110.246.13:9200/test/_analyze

{
    "text": "中华人民共和国的人民万岁",
    "tokenizer": "ik_max_word"
}
~~~



[ik分词地址](https://github.com/medcl/elasticsearch-analysis-ik/releases)



