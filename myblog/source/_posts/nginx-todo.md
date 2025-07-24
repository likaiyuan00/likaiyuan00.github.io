---
title: nginx_todo
date: 2025-07-21 14:19:19
tags: 
categories: 中间件
---
# proxy_pass转发策略
## 请求url和转发一致
```
后端服务实际处理路径为 /api/upload，与客户端请求路径一致。
Nginx配置：


location /api/ {         # 匹配客户端请求中的 /api/ 前缀
    proxy_pass http://backend;  # 不改变路径，直接转发 /api/xxx 到后端
}

转发效果：
客户端请求 → /api/upload
Nginx转发 → http://backend/api/upload
```

## 后端服务需要基础路径（去掉/api/前缀）
```
后端路由示例：
后端服务处理根路径 /upload，不需要 /api/ 前缀。
Nginx配置：
nginx


location /api/ {
    # 通过 rewrite 移除 /api/ 前缀
    rewrite ^/api/(.*) /$1 break;  
    proxy_pass http://backend;  
}
或

location /api/ {
    # 直接在 proxy_pass 中追加路径
    proxy_pass http://backend/;  # 注意结尾的斜杠
}
● 转发效果：
客户端请求 → /api/upload
Nginx转发 → http://backend/upload
```

## 转发加url注意点
```
curl http://127.0.0.1/api/client-test
location /api/ {
    proxy_pass http://backend/test/;  # 结尾必须加斜杠
}
127.0.0.1- - [21/Jul/2025:14:32:45 +0800] "GET /test/client-test HTTP/1.0" 404 178 "-" "curl/7.58.0" "-"Proxy: http://-
127.0.0.1- - [21/Jul/2025:14:32:45 +0800] "GET /api/client-test HTTP/1.1" 404 178 "-" "curl/7.58.0" "-"Proxy: http://127.0.0.1
客户端请求 /api/client-test → 后端路径 /test/client-test



curl http://127.0.0.1/api-test/client-test
location /api-test/ {
    proxy_pass http://backend/test;  # 无斜杠 → 路径合并
}
127.0.0.1- - [21/Jul/2025:14:32:28 +0800] "GET /testclient-test HTTP/1.0" 404 178 "-" "curl/7.58.0" "-"Proxy: http://-
127.0.0.1- - [21/Jul/2025:14:32:28 +0800] "GET /api-test/client-test HTTP/1.1" 404 178 "-" "curl/7.58.0" "-"Proxy: http://127.0.0.1

客户端请求: http://example.com/api-test/client-test
实际转发路径: http://backend/testclient-test

```

# 日志记录
## 常用内置变量
```
$http_ 变量可以自定义比如$http_lky 请求头里面有lky:value则$http_lky等于value
$args ： #这个变量等于请求行中的参数，同$query_string
$content_length ： # 请求头中的Content-length字段。
$content_type ： # 请求头中的Content-Type字段。
$document_root ： # 当前请求在root指令中指定的值。
$host ： # 请求主机头字段，否则为服务器名称。
$http_user_agent ：#  客户端agent信息
$http_cookie ： # 客户端cookie信息
$limit_rate ： # 这个变量可以限制连接速率。
$status  # 请求状态
$body_bytes_sent # 发送字节
$request_method ： # 客户端请求的动作，通常为GET或POST。
$remote_addr ： # 客户端的IP地址。
$remote_port ： # 客户端的端口。
$remote_user ： # 已经经过Auth Basic Module验证的用户名。
$request_filename ： # 当前请求的文件路径，由root或alias指令与URI请求生成。
$scheme ： # HTTP方法（如http，https）。
$server_protocol ： # 请求使用的协议，通常是HTTP/1.0或HTTP/1.1。
$server_addr ： # 服务器地址，在完成一次系统调用后可以确定这个值。
$server_name ： # 服务器名称。
$server_port ： # 请求到达服务器的端口号。
$request_uri ： # 包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。
$uri ： # 不带请求参数的当前URI，$uri不包含主机名，如”/foo/bar.html”。
$document_uri ： # 与$uri相同。

```
## 日志配置变量
```
location / {
      proxy_pass [$Domain]; #必须
      index index.html index.htm index.jsp index.shtml;
      proxy_redirect off;
      proxy_set_header Host $host;
      proxy_set_header Lky $remote_addr;
      proxy_set_header REMOTE-HOST $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-My-Header "My Value";
      #在日志中使用$http_x_my_header就可以获取到值，可以写死也可以用内置变量比如set自定义
      经测试 proxy_set_header 第一个字母必须大写，只能用-不能用_
      log_format 日志格式必须是$http开头-需要换成_而且必须全部小写
    }

log_format custom '$remote_addr [$time_local] "$request" '
                  'Lky:$http_lky' ;
access_log /var/log/nginx/access.log custom;

#如果是  proxy_pass http://127.0.0.1:83 第一跳记录上游日志包含真实ip，第二条是客户端访问的不包含Lky，一个请求有两个日志输出
#127.0.0.1 - - [23/May/2025:16:31:41 +0800] "GET / HTTP/1.0" 200 4833 "-" "curl/7.60.0" "10.0.1.100" 10.0.1.100Lky:"10.0.1.100"
#这里的日志可以通过proxy_set_header自定义，比如获取客户端真实IP

#10.0.1.100 - - [23/May/2025:16:31:41 +0800] "GET / HTTP/1.1" 200 4833 "-" "curl/7.60.0" "-" 10.0.1.100:83Lky:"-"
#这个则是客户端直接请求的日志，因为$remote_addr为空

#测试响应头
add_header Lky $remote_addr always;
GET / HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Lky: 192.168.1.100
```

