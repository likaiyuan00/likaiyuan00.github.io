---
title: mq
date: 2025-08-22 15:28:18
tags:
categories: 中间件
---
# rabbitMQ
* 生产者路由  vhost  >  exchange  >  queue
每一个vhost本质上是一个mini版的RabbitMQ服务器，拥有自己的交换机、队列、绑定等，拥有自己的权限机制
```yml
version: '3'
services:
  rabbitmq:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/mq:rabbit-3-management
    container_name: rabbitmq
    restart: always
    ports:
      - "5672:5672"
      - "15672:15672"
      - "15692:15692"  # Prometheus 监控指标端口（可选）
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: secret
      RABBITMQ_DEFAULT_VHOST: /prod  # 指定默认虚拟主机
      ABBITMQ_NODE_IP_ADDRESS: 0.0.0.0
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq  # 持久化数据
      - rabbitmq_logs:/var/log/rabbitmq  # 持久化日志
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "status"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  rabbitmq_data:
  rabbitmq_logs:
```
* 简单使用demo
```python
#生产者
import pika


credentials = pika.PlainCredentials('admin', 'secret')
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='47.109.136.x', port=5672, virtual_host='/prod', credentials=credentials)
)
channel = connection.channel()

channel.queue_declare(queue='test_queue')
channel.basic_publish(exchange='', routing_key='test_queue', body='Hello from Python!')
connection.close()

#消费者
import pika

def callback(ch, method, properties, body):
    print(f"Received: {body.decode()}")

credentials = pika.PlainCredentials('admin', 'secret')
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='47.109.136.x', port=5672, virtual_host='/prod', credentials=credentials)
)
channel = connection.channel()

channel.queue_declare(queue='test_queue')
channel.basic_consume(queue='test_queue', on_message_callback=callback, auto_ack=True)
print('Waiting for messages...')
channel.start_consuming()

```
* 关于回调函数
```python
class Library:
    def __init__(self):
        self.callback = None

    def register_callback(self, callback_func):
        self.callback = callback_func  # 保存函数对象

    def trigger_event(self):
        if self.callback:
            # 在事件发生时调用回调函数，并传入参数
            self.callback("param1", "param2")

# 用户定义的函数
def my_callback(a, b):
    print(f"回调触发: {a}, {b}")

# 使用库
lib = Library()
lib.register_callback(my_callback)  # 传递函数对象
lib.trigger_event()  # 输出: 回调触发: param1, param2
```




# kafka
* 生产者路由  group  >   topic   >  tag,适合吞吐量很大的场景比如大数据
```yml
version: '3'
services:
  zookeeper:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/mq:zk-3.8  # Zookeeper 镜像
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes  # 允许匿名访问（测试用）
    networks:
      - kafka-net
    volumes:
      - zookeeper_data:/bitnami/zookeeper

  kafka:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/mq:kafka-3.4   # Kafka 镜像
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes  # 允许明文监听
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092  # 客户端访问地址
    depends_on:
      - zookeeper
    networks:
      - kafka-net
    volumes:
      - kafka_data:/bitnami/kafka

networks:
  kafka-net:
    driver: bridge

volumes:
  zookeeper_data:
  kafka_data:
```
```shell
# 创建主题 "test-topic"
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --replication-factor 1 \
  --partitions 1 \
  --topic test-topic
kafka-topics.sh    --bootstrap-server localhost:9092   --list
# 启动生产者，发送消息
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic

# 启动消费者，接收消息
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --from-beginning

# 不同节点的配置差异（以 101 节点为例）
broker.id=1  # 102 节点改为 2，103 节点改为 3
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://192.168.1.101:9092  # 修改为当前节点 IP

# 所有节点相同配置
zookeeper.connect=192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181
log.dirs=/data/kafka/logs
num.partitions=3  # 默认分区数（建议 >= Broker 数量）
default.replication.factor=3  # 默认副本数（建议 = Broker 数量）
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
delete.topic.enable=true
```
















# 
