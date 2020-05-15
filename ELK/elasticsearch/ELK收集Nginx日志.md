### 1.配置nginx日志
编辑nginx.conf文件  `vim /etc/nginx/nginx.conf` 在http节点下配置如下
```
log_format json '{"@timestamp":"$time_iso8601",'
                           '"@version":"1",'
                           '"client":"$remote_addr",'
                           '"url":"$uri",'
                           '"status":"$status",'
                           '"domain":"$host",'
                           '"host":"$server_addr",'
                           '"size":$body_bytes_sent,'
                           '"responsetime":$request_time,'
                           '"referer": "$http_referer",'
                           '"ua": "$http_user_agent"'
               '}';
    access_log /data/nginx/logs/access_json.log json;
```
目的就是将nginx的日志以json的形式进行文件存储，方便es存储
访问nginx 查看日志 `tail -f /data/nginx/logs/access_json.log` 可以看到新的入职信息说明配置正常
![](https://upload-images.jianshu.io/upload_images/8387919-80d53823eaf1547a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 安装elk
采用`docker-compose`安装
```
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


  logstash:
    image: "logstash:7.1.1"
    container_name: "logstash"
    restart: "always"
    networks:
      - "elk"
    ports:
      - "5044:5044"
      - "9600:9600"
    volumes:
      - "logstash:/usr/share/logstash"
      - "/data/nginx/logs:/data/nginx/logs"  

networks:
  elk:

volumes:
  elasticsearch:
  logstash:
  kibana:

```
- 配置logstash.yml
在`config/logstash.yml`文件下追加
```
path.config: /usr/share/logstash/conf.d/*.conf
```
- 配置logstash日志处理文件
新增`conf.d/logstash.conf`文件 内容如下：
```
input {
   file {
        type => "nginx-access-log"
        path => "/data/nginx/logs/access_json.log"
        start_position => "beginning"
        stat_interval => "2"
        codec => json
   }

}
filter {}
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    #index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    index => "logstash-nginx-access-log-%{+YYYY.MM.dd}"
    #user => "elastic"
    #password => "changeme"
  }
  stdout {
        codec => json_lines
  }
}
```
说明：
- start_position 指从文件开始位置读取

- stat_interval 指每间隔两秒读取一次

- index 指定索引名称

- user | password 这里没有安装`xpack`插件，所以用户名，密码不用配置，如果需要可以 自行配置
  启动`docker-compose` 
  `docker -compose up -d --build`

之后打开head插件发现发出来一个index库
  ![](https://upload-images.jianshu.io/upload_images/8387919-71158619af1b7284.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开`http:{host}:5601`在kibana中添加nginx日志匹配规则

![](https://upload-images.jianshu.io/upload_images/8387919-1d884cb3e365a124.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Management-->index patterns-->create index pattern
输入`logstash-nginx-*` 就是在logstash中配置的索引名称前缀
然后配置时间排序字段 `@timestamp ` 这样kibana就可以根据此字段进行时间倒序展示了
配置好之后就可以在左侧`discover`中查看对应的日志索引信息了
![](https://upload-images.jianshu.io/upload_images/8387919-8ffdf421a45df77c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
另外可以进行字段筛选显示
![](https://upload-images.jianshu.io/upload_images/8387919-3cf48c7bed6a0e9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

