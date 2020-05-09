### docker-compose搭建ELK环境

![](https://i1.pickpik.com/photos/718/586/394/banff-national-park-a-preview.jpg)

#### 环境信息

- CentOS 7.4 系统
- Docker version 18.06.1-ce
- docker-compose version 1.22.0
- 部署单节点 ELK

#### 参数配置

在宿主机执行
```shell
# 设置内核参数
sysctl -w vm.max_map_count=262144
# 生效设置
sysctl -p
# 重启 docker，让内核参数对docker服务生效
systemctl restart docker
```
- 原因分析:

`vm.max_map_count`参数，是允许一个进程在VMAs拥有最大数量（VMA：虚拟内存地址， 一个连续的虚拟地址空间），当进程占用内存超过时， 直接OOM。

elasticsearch占用内存较高。官方要求max_map_count需要配置到最小262144。

max_map_count配置文件写在系统的/proc/sys/vm中

通过docker inspect命令， 可查看docker使用宿主机的/proc/sys作为只读路径之一

```json
     "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
```

说明镜像使用宿主机的max_map_count参数。因此直接修改宿主机的max_map_count参数即可

#### docker-compose文件

```yml
version: "3"
services:
  elasticsearch:
    image: "elasticsearch:6.7.1"
    container_name: "elasticsearch"
    restart: "always"
    volumes:
      - "elasticsearch_data:/usr/share/elasticsearch"
    #vim /etc/sysctl.conf
    #vm.max_map_count=262144
    #sysctl -w vm.max_map_count=262144
    #sysctl -p
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    networks:
      - "elk"
    ports:
      - "9200:9200"
      - "9300:9300"

  kibana:
    image: "kibana:6.7.1"
    container_name: "kibana"
    restart: "always"
    volumes:
      - "kibana:/usr/share/kibana"
    networks:
      - "elk"
    ports:
      - "5601:5601"


#  logstash:
#    image: "logstash:6.7.1"
#    container_name: "logstash"
```

> 这里我暂时没有用到logstash，其中的版本可以根据情况自行升级，但是需要注意版本一致或兼容，具体请`elastic`参考官网

- 启动：`docker-compose up -d`
- 查看日志 `dcoker-compose logs -f`
- 修改配置后重新构建 `docker-compose up -d --build`
- 停止服务 `docker-compose stop kibana`
- 删除所有docker-compose.yml中描述的服务 `docker-compose down`

#### 修改elasticsearch.yml

```properties
cluster.name: "docker-cluster"
network.host: 0.0.0.0
http.cors.enabled: true
http.cors.allow-origin: "*"
```

其中后两行是允许跨域访问

- 重新构建 ` dockr-compose up -d --build`

- 访问 http://{host}:9200/

![image-20200506101750385](C:\Users\haopeng\AppData\Roaming\Typora\typora-user-images\image-20200506101750385.png)

- 访问head插件

> 这里我使用的是chrom中的head插件，也可以自行安装head插件 [github](https://github.com/mobz/elasticsearch-head)

![image-20200506101940711](C:\Users\haopeng\AppData\Roaming\Typora\typora-user-images\image-20200506101940711.png)



- 访问kibana 

![](https://upload-images.jianshu.io/upload_images/8387919-5ebde71cc89a8c3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



之后就可以在kibana中的devtools中进行es的查询了。



#### 参考

- [elastic官网](https://www.elastic.co/cn/)
- [head插件](https://github.com/mobz/elasticsearch-head)