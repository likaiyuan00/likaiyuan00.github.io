---
title: tcp
date: 2025-07-25 15:36:08
tags:
categories: linux
---

# 三握四挥
![alt text](image.png)

```
 （1）第一次握手：Client将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给Server，Client进入SYN_SENT状态，等待Server确认。

  （2）第二次握手：Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK都置为1，ack=J+1，随机产生一个值seq=K，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。

  （3）第三次握手：Client收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给Server，Server检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED状态，完成三次握手，随后Client与Server之间可以开始传输数据了

客户端发起fin位为1的FIN报文，此时客户端进入FIN_WAIT_1状态
服务端接受到FIN 报文后，发送ack应答报文，此时服务端进入close_wait状态
客户端接受到ack应答报文后，进入FIN_WAIT_2状态
服务端处理完数据后，向客户端发送FIN报文，此时服务端进入LAST_ACK状态
客户端接受到FIN报文后，客户端发送应答ack报文，进入TIME_WAIT阶段
服务端接受到ack报文后，断开连接，处于close状态
客户端过一段时间后，也就是2MSL后，进入close状态
```




# Time Wait
## 概念
* 谁先关闭谁最后进入timewait状态，time_wait 状态下，TCP 连接占用的端口，无法被再次使用
```
close 短连接
每个HTTP请求都需要重新完成TCP三次握手建立连接，数据传输完成后四次挥手关闭连接
keep-alive 长连接
在HTTP1.1协议中默认长连接，有个 Connection 头，Connection有两个值，close和keep-alive，这个头就相当于客户端告诉服务端，服务端你执行完成请求之后，是关闭连接还是保持连接，保持连接就意味着在保持连接期间，只能由客户端主动断开连接。还有一个keep-alive的头，设置的值就代表了服务端保持连接保持多久
#ngx
keepalive 100;          # 保持的空闲连接数
keepalive_timeout 60s;   # 空闲超时时间
keepalive_requests 100;  # 单连接最大请求数
location / {
    proxy_pass http://backend_servers;
    proxy_http_version 1.1;     # 关键：使用 HTTP/1.1
    proxy_set_header Connection "";  # 清除 Connection 头（避免传递错误值）
}
# Linux 内核参数（默认值通常为 7200 秒）默认不启用
sysctl -w net.ipv4.tcp_keepalive_time=1800  # 空闲 1800 秒后发送探针
sysctl -w net.ipv4.tcp_keepalive_probes=3   # 发送 3 次无响应后关闭
sysctl -w net.ipv4.tcp_keepalive_intvl=15   # 每次探针间隔 15 秒
```
## 相关参数
```
netstat -ant | awk '/^tcp/ {++y[$NF]} END {for(w in y) print w, y[w]}'
net.ipv4.tcp_syncookies = 1 
net.ipv4.tcp_tw_reuse = 1 
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
==============================================
net.ipv4.tcp_tw_reuse = 1
表示开启重用。允许将一个处于TIME-WAIT状态的端口重新用于新的TCP连接，默认为0，表示关闭，其防止重复报文的原理也是时间戳
net.ipv4.tcp_tw_recycle = 1
表示开启TCP连接中TIME-WAIT sockets的快速回收，意思就是系统会保存最近一次该socket连接上的传输报文（包括数据或者仅仅是ACK报文）的时间戳，当相同四元组socket过来的报文的时间戳小于缓存下来的时间戳则丢弃该数据包，并回收这个socket，默认为0，表示关闭。开启这个功能风险有点大，NAT环境可能导致DROP掉SYN包（回复RST），在NAT场景下不要使用。需要注意在Linux内核4.10版本以后该参数就已经被移除了。
net.ipv4.tcp_fin_timeout = 60
这个时间不是修改2MSL的时长，主动关闭连接的一方接收到ACK之后会进入，FIN_WAIT-2状态，然后等待被动关闭一方发送FIN，这个时间是设置主动关闭的一方等待对方发送FIN的最长时长，默认是60秒。在这个状态下端口是不可能被重用的，文件描述符和内存也不会被释放，因为这个阶段被动关闭的一方有可能还有数据要发送，因为对端处于CLOSE_WAIT状态，也就是等待上层应用程序。关于这个的真实含义我希望大家清楚，而且不要调整的太小当然太大也不行，至少在3.10内核版本上这个参数不是调整的TIME_WAIT时长。
net.ipv4.ip_local_port_range = 32768 60999
表示用于外连使用的随机高位端口范围，也就是作为客户端连接其他服务的时候系统从这个范围随机取出一个端口来作为源端口使用来去连接对端服务器，这个范围也就决定了最多主动能同时建立多少个外连。
net.ipv4.tcp_max_tw_buckets = 6000
同时保持TIME_WAIT套接字的最大个数，超过这个数字那么该TIME_WAIT套接字将立刻被释放并在/var/log/message日志中打印警告信息（TCP: time wait bucket table overflow）。这个过多主要是消耗内存，单个TIME_WAIT占用内存非常小，但是多了就不好了，这个主要看内存以及你的服务器是否直接对外。
使用net.ipv4.tcp_tw_reuse和net.ipv4.tcp_tw_recycle 的前提是开启时间戳net.ipv4.tcp_timestamps = 1不过这一项默认是开启的
```

# CLOSE_WAIT
* 这种状态的含义其实是表示在等待关闭。怎么理解呢？当对方close一个SOCKET后发送FIN报文给自己，你系统毫无疑问地会回应一个ACK报文给对方，此时则进入到CLOSE_WAIT状态。接下来呢，实际上你真正需要考虑的事情是查看你是否还有数据发送给对方，如果没有的话，那么你也就可以close这个SOCKET，发送FIN报文给对方，也即关闭连接。所以你在CLOSE_WAIT状态下，需要完成的事情是等待你去关闭连接。CLOSE_WAIT一般是由于对端主动关闭，而我方没有正确处理的原因引起的，临时解决重启服务，永久解决就是修改程序逻辑
```
客户端（主动关闭方）          服务器（被动关闭方）
       |                               |
       |--- GET /bigfile.zip --------->| 
       |<--- 200 OK + 文件数据 ---------|
       |                               |
       |--- FIN (我要关闭) ------------>| → 客户端进入 FIN_WAIT_1
       |<--- ACK ----------------------| → 服务端进入 CLOSE_WAIT
       |                               |（此时服务端还在发送剩余文件数据）
       |<--- 剩余数据包 ----------------|
       |<--- FIN (我也关闭) ------------| → 服务端进入 LAST_ACK
       |--- ACK ----------------------->| → 客户端进入 TIME_WAIT                
#没有调用s.close()关闭socket，会造成大量CLOSE_WAIT
import socket
import time


def create_sockets(num_sockets):
    sockets = []
    for _ in range(num_sockets):
        # 创建一个 TCP 套接字
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        print(f"创建 socket {_ + 1}: {s.fileno()}，状态为 alloc")
        sockets.append(s)
    return sockets

if __name__ == "__main__":
    num_sockets = 10
    while True:
        num_sockets  += 10
        sockets = create_sockets(num_sockets)
        print(f"总共创建了 {num_sockets} 个 socket 对象。")
        time.sleep(10)

#shell
cat /proc/net/sockstat | grep sockets | awk '{print $3}'
netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'
for i in `ls /proc/ |grep '[0-9]'`
do
    mycount=`ls /proc/$i/fd|wc -l`
    echo "$i $mycount"
done
```
