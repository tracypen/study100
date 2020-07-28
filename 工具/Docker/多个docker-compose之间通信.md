> 多个docker-compose之间的通信

- 1.elk服务`docker-compose`文件

> 在最后声明一个`nwtworks`网络`elk`，这样运行，启动dockr-compose后会自动创建对应的网络，也可以使用`docker network create ***`来创建

```yaml
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
  cerebro:
    image: "lmenezes/cerebro"
    restart: "always"
    container_name: "cerebro"
    ports: 
      - "9000:9000"
networks:
  elk:

volumes:
  elasticsearch:
  kibana:
```

> 在需要连接通信的`docker-compose`中，文件最后声明networks，主要这里的名称是`elk_elk`,使用`docker network ls`

- 2.基础服务`docker-compose`文件

```yaml
version: '3'
services:
  mysql:
    image: mysql:5.7.22
    restart: always
    privileged: true
    container_name: mysql-5.7
    volumes:
      - ./mysql/data:/var/lib/mysql
      - ./mysql/config/my.cnf:/etc/my.cnf
      - ./mysql/init:/docker-entrypoint-initdb.d/
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: root
    ports:
      - '3306:3306'
  redis:
    image: redis:4.0
    restart: always
    container_name: redis
    ports:
      - 6379:6379
    volumes:
      - ./redis/data:/data
    command: redis-server
  sjy-api:
    image: sjy-api:latest
    restart: always
    container_name: sjy-api
    environment:
      - JVM_OPTS=-XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m -Xms512m -Xmx1024m -Xmn256m -Xss256k -XX:SurvivorRatio=8 -XX:+UseConcMarkSweepGC
    volumes:
      - ./sjy/logs:/logs
      - /etc/localtime:/etc/localtime
    ports:
      - 5050:5050
    links:
      - mysql
      - redis
    external_links:
      - elasticsearch
    depends_on:
      - mysql
      - redis
networks:
  default:
    external:
     name: elk_elk
```

- 使用`docker-compose up -d --build`启动，进入容器内部检查网络情况

```
root@zssy-test:/opt/docker/base# docker exec -it sjy-api /bin/sh
/ # ping elasticsearch
PING elasticsearch (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.137 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.117 ms
64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.110 ms
64 bytes from 172.18.0.2: seq=3 ttl=64 time=0.098 ms
64 bytes from 172.18.0.2: seq=4 ttl=64 time=0.106 ms
64 bytes from 172.18.0.2: seq=5 ttl=64 time=0.107 ms

```


> 这样就可以使用如下配置去进行容器间的访


![](https://upload-images.jianshu.io/upload_images/8387919-6fd9cf7af05ba216.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)