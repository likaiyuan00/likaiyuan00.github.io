---
title: proxysql
date: 2025-08-19 17:15:15
tags:
categories: 中间件
---
* proxysql是一款代理数据库的开源软件，此外还有maxsacle<br>
https://github.com/sysown/proxysql

# 搭建读写分离
```shell
mkdir -p proxysql && cd proxysql
mkdir -p mysql/{master,proxysql,slave}/conf


#proxysql.cnf配置
datadir="/var/lib/proxysql"

mysql_servers =
(
    {
        address = "mysql-master"
        port = 3306
        hostgroup = 10  # 写组
        max_connections = 200
    },
    {
        address = "mysql-slave"
        port = 3306
        hostgroup = 20  # 读组
      #  weight = 110
        max_connections = 1000
    },
    #  {
  #      address = "mysql-slave2"   # 从库2
  #      port = 3306
  #      hostgroup_id = 20
   #     weight = 100
  #      max_connections = 500
   # }
)

#CREATE USER 'appuser'@'%' IDENTIFIED BY 'AppUser123!';
#GRANT ALL PRIVILEGES ON *.* TO 'appuser'@'%';
mysql_users =
(
    {
        username = "appuser" #后端mysql的用户
        password = "AppUser123!"
        default_hostgroup = 10
    }
)

mysql_query_rules =
(
    {
        rule_id = 100 #规则唯一标识符，数值越小优先级越高
        active = 1  #规则是否启用：1 启用，0 禁用
        match_pattern = "^SELECT" #匹配读请求
        destination_hostgroup = 20 #匹配的 SQL 请求路由到的主机组
        apply = 1 #是否在匹配后终止后续规则匹配：1 终止，0 继续
    },
    {
        rule_id = 200
        active = 1
        match_pattern = "^((?!SELECT).)*$" 
        destination_hostgroup = 10
        apply = 1
    }
)
#docker exec -it proxysql-proxysql-1 mysql -uadmin -padmin -h127.0.0.1 -P6032
#6032是管理端口，显示的是proxysql的元数据表，
#可以通过环境变量修改默认密码admin
#UPDATE global_variables SET variable_value='admin:new_password' WHERE variable_name='admin-admin_credentials';
#SELECT * FROM global_variables  WHERE variable_name LIKE 'admin-admin_%';


#mysql -uappuser -pAppUser123! -h127.0.0.1 -P6033 
#6033是服务端口，直接可以路由到后端mysql

#master配置
[mysqld]
server_id = 1
log_bin = mysql-bin
binlog_format = ROW

#slave配置
[mysqld]
server_id = 2
relay_log = mysql-relay-bin
read_only = 1

#======================================================================================
cat >> docker-compose.yaml << 'EOF'
version: '3.8'

services:
  mysql-master:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/mysql:8.0.43
    networks:
      - mysql-network
    environment:
      MYSQL_ROOT_PASSWORD: MasterRoot123!
      MYSQL_REPLICATION_USER: repl
      MYSQL_REPLICATION_PASSWORD: ReplPass123!
    volumes:
      - ./mysql/master/conf:/etc/mysql/conf.d
      - ./mysql/master/data:/var/lib/mysql
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci

  mysql-slave:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/mysql:8.0.43
    networks:
      - mysql-network
    environment:
      MYSQL_ROOT_PASSWORD: SlaveRoot123!
    volumes:
      - ./mysql/slave/conf:/etc/mysql/conf.d
      - ./mysql/slave/data:/var/lib/mysql
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci

  proxysql:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/mysql:proxysql
    networks:
      - mysql-network
    ports:
      - "6033:6033"
      - "6032:6032"
    volumes:
      - ./mysql/proxysql/proxysql.cnf:/etc/proxysql.cnf
    depends_on:
      - mysql-master
      - mysql-slave

networks:
  mysql-network:
    driver: bridge
EOF

#===================================================================================
#配置主从复制
#8.4版本以上语法 SHOW BINARY LOG STATUS;
docker exec -it proxysql-mysql-master-1 mysql -uroot -p"MasterRoot123!" -e \
"CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'ReplPass123!'; \
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%'; \
FLUSH PRIVILEGES; \
SHOW MASTER STATUS;"

#MASTER_LOG_POS可以通过 SHOW MASTER STATUS;在master节点执行查看
docker exec -it proxysql-mysql-slave-1 mysql -uroot -pSlaveRoot123! -e \
"CHANGE MASTER TO \
MASTER_HOST='mysql-master', \
MASTER_USER='repl', \
MASTER_PASSWORD='ReplPass123!', \
MASTER_LOG_FILE='mysql-bin.000003', \
MASTER_LOG_POS=827; \
START SLAVE; \
SHOW SLAVE STATUS\G"


#SHOW MASTER STATUS;
#SHOW SLAVE STATUS;
#SHOW BINARY LOGS;
#SHOW BINLOG EVENTS;
#show binlog events in 'mysql-bin.000002' from 219 limit 5;

#reset master; #清空所有binlog
#flush logs;  #刷新binlog，直接生成新的binlog文件

#SHOW VARIABLES LIKE "%expire_logs_days%";    #binlog保存多久 
#SHOW VARIABLES LIKE 'binlog_expire_logs_seconds'; 优先级比expire_logs_days高
#SHOW VARIABLES LIKE "%max_binlog%";  #单个binlog到多大开始生成新的

#================================================================================
#创建proxysql的监控后端mysql用户，注意只需要在master节点执行，因为已经主从同步了，不然主从会报错停止
-- 创建监控用户
CREATE USER 'monitor'@'%' IDENTIFIED  BY 'MonitorPass123!';

-- 授予必要权限（ProxySQL 健康检查需要）
GRANT USAGE, REPLICATION CLIENT ON *.* TO 'monitor'@'%';

-- grant all privileges on *.* to 'monitor'@'%' with grant option;

-- 刷新权限
FLUSH PRIVILEGES;

#登录proxysql
docker exec -it proxysql-proxysql-1 mysql -uadmin -padmin -h127.0.0.1 -P6032
#proxysql监控用户名和密码默认都是monitor，可以通过以下语句查看
SELECT * FROM global_variables 
WHERE variable_name IN ('mysql-monitor_username', 'mysql-monitor_password');
#修改密码
UPDATE global_variables SET variable_value='MonitorPass123!'
WHERE variable_name='mysql-monitor_password';
load mysql variables to runtime;
save mysql variables to disk;

#proxysql的相关元数据表
SELECT hostgroup_id, hostname, status FROM main.mysql_servers;;
hostgroup_id：服务器所属主机组（如读写分离的读组 20 和写组 10）。
hostname：服务器地址（IP 或域名）。
status：节点状态，如 ONLINE（正常）、OFFLINE_SOFT（软下线）、OFFLINE_HARD（硬下线）、SHUNNED（临时屏蔽）


SELECT hostgroup, username, digest_text,count_star,sum_time FROM stats.stats_mysql_query_digest;
digest_text：归一化后的 SQL 模板（如 SELECT * FROM users WHERE id=?）。
count_star：该 SQL 模板的执行次数。
sum_time：该 SQL 模板的总耗时（微秒）。
hostgroup：请求路由到的主机组。
username：执行 SQL 的客户端用户。


SELECT * FROM global_variables WHERE variable_name LIKE 'mysql-monitor%';
mysql-monitor_username：监控用户（需在后端 MySQL 存在并授权）。
mysql-monitor_ping_interval：Ping 检查间隔（毫秒）。
mysql-monitor_read_only_interval：检查主库只读状态的频率。

SELECT * FROM monitor.mysql_server_connect_log   ORDER BY time_start_us DESC           LIMIT 3;
connect_error：失败原因（如超时、权限拒绝）。
time_start_us：连接开始时间（微秒精度）。

```

  

# 主从同步如果报错
```shell
#主从错误，在主库创建完用户从库就不用创建了，不然冲突主从就停止了
#可以通过以下命令查询导致主从停止的语句，并且修改
SELECT * FROM performance_schema.replication_applier_status_by_worker\G


#重新启动
STOP REPLICA;
START REPLICA;

```
