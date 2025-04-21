---
title: elfk部署使用
date: 2025-04-18 09:56:59
tags:
categories: 中间件
---

>filebeat不建议容器启动，适合放到每个节点采集日志统一发给logstash；如果全部输出到elasticsearch会导致负载比较高；不建议每个节点用logstash采集因为比较重，filebeat比较轻量级

# 安装elfk
```shell
curl -SL https://github.com/docker/compose/releases/download/v2.30.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
#将可执行权限赋予安装目标路径中的独立二进制文件
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

yum install -y https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.3.2-x86_64.rpm

cat >> ./elk.yml << EOF
version: '3.8'
services:
  elasticsearch:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/elasticsearch:7.14.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node  # 单节点模式
      - ES_JAVA_OPTS=-Xms512m -Xmx512m  # JVM 堆内存限制
      - ELASTIC_PASSWORD=Ytest@123  # 设置 Elasticsearch 密码
    volumes:
      - ./elasticsearch/data:/usr/share/elasticsearch/data  # 数据持久化
#      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml  # 自定义配置（可选）
    ports:
      - "9200:9200"  # REST API
      - "9300:9300"  # 集群通信
    networks:
      - elk
  logstash:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/logstash:7.14.0
    container_name: logstash
    volumes:
      - ./logstash/config/logstash.conf:/usr/share/logstash/pipeline/logstash.conf  # 自定义 Logstash 管道配置
      - ./logstash/logs:/usr/share/logstash/logs  # 日志持久化
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m  # JVM 堆内存限制
    ports:
      - "5044:5044"  # Beats 输入端口（如 Filebeat）
      - "5000:5000/tcp"  # TCP 输入
      - "5000:5000/udp"  # UDP 输入
    depends_on:
      - elasticsearch
    networks:
      - elk
  kibana:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/kibana:7.14.0
    container_name: kibana
    environment:
      - I18N_LOCALE=zh-CN
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200  # 指向 Elasticsearch 服务
      - ELASTICSEARCH_USERNAME=elastic  # 默认用户名
      - ELASTICSEARCH_PASSWORD=Ytest@123  # 与 Elasticsearch 密码一致
    ports:
      - "5601:5601"  # Kibana Web 界面
    depends_on:
      - elasticsearch
    networks:
      - elk
networks:
  elk:
    driver: bridge
EOF
mkdir ./logstash/config -p
cat >> ./logstash/config/logstash.conf << EOF
# ./logstash/config/logstash.conf
input {
  tcp {
    port => 5000  # 监听 TCP 日志
  }
  beats {
    port => 5044  # 接收 Filebeat 输入
  }
}
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }  # 解析 Apache 日志
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]  # 时间解析
  }
}
output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    user => "elastic"
    password => "Ytest@123"
    index => "logs-%{+YYYY.MM.dd}"  # 按日期创建索引
  }
}
EOF
chmod 777 elasticsearch/data
```
# filebeat
**根据不同tag写入不同的logstash后续分割和输出建立索引好区分**
```
filebeat.inputs: # filebeat input输入
- type: log    # 标准输入
  enabled: true  # 启用标准输入
  paths:
    - /var/log/*
  tags: ["system"]
  #  fields:
  #    type: "system_log"
- type: filestream
  paths:
    - "/var/log/nginx/*.log"
  tags: ["nginx"]   # 标记为 nginx 日志
#output.console:
# enabled: true               # 启用控制台输出
  #  pretty: true                # 美化 JSON 格式
  # codec.json:
  #   pretty: true
  # escape_html: false        # 不转义 HTML 符号（保持原始格式）
 
# 输出到 Logstash - 用于生产数据处理
output.logstash:
  enabled: true               # 启用 Logstash 输出
  #  when:
  #    equals:
  #      fields.type: "system_log"
  hosts: ["127.0.0.1:5044"]  # Logstash 的地址和端口（支持多个主机负载均衡）
  when.contains:
      tags: "system"  # 匹配 tags 包含 "system"
  hosts: ["127.0.0.1:5045"]
  enabled: true
  when.contains:
    tags: "nginx"  # 匹配 tags 包含 "nginx"
```
# logstash
**根据不同type进行过滤和输出索引**
```
Logstash Reference [7.10] | Elastic

input {
  tcp {
    port => 5000  # 监听 TCP 日志
  }
  beats {
    port => 5044  # 接收 Filebeat 输入
    type => "system"
  }
  beats {
    port => 5045  # 接收 Filebeat 输入
    type => "nginx"
  }
}
  
 
filter {
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]  # 时间解析
  }
 
  if[type] == "nginx" {
    grok {
      match => { "message" => "%{HTTPD_COMMONLOG}" }  # 解析 nginx 日志,如果不区分；system类型是解析不了的，会直接报错
      remove_field => ["@version"]
     }
  }
  #对于system类型可以再写个if来单独过滤
  if[type] == "system" {
    grok {
      match =>  {"message" => "%{IPV4:ip}"}  
      remove_field => ["@version"]
     }
    mutate {  #这里过滤器乱写的，需要根据自身的业务配置
        remove_field => ["timestamp"]
        gsub => ["message","\s","| "]
        split => ["message","|"]
        replace => { "timenew" =>  "%{+yyyy-MM-dd}" }
        add_field => {
         "year" => "%{+yyyy}"
         "month" => "%{+MM}"
         "day" => "%{+dd}"
         "status" => "%{[message][1]}"
         "code" => "%{[message][2]}"
        }
    }
  }
 
  
}
#必须通过type指定不同输出创建不同的index =>,否则index的字段不一样，当第一个index结构确定后，第二个输入无法输出到第一个index，因为字段不一样
output {
  if "system" in [tags] {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      user => "elastic"
      password => "Ytest@123"
      index => "filebeat-system-logs-%{+YYYY.MM.dd}"  # 按日期创建索引
    }
  }  
  if "nginx" in [tags] {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      user => "elastic"
      password => "Ytest@123"
      index => "filebeat-nginx-logs-%{+YYYY.MM.dd}"  # 按日期创建索引
    }
  }  
}
```
# elasticsearch
>常用语法
>>/_cat <br>
/_cat/master?help<br>
/_cat/indices?v  显示title<br>
/_cat/indices<br>
logs-2025.03.24 为索引名称<br>
/logs-2025.03.24/_search 查看文档<br>
/logs-2025.03.24/ 查看索引结构<br>
/logs-2025.03.24/_doc/_search?q=message:test
