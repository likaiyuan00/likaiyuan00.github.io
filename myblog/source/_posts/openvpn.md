---
title: openvpn
date: 2025-04-21 21:11:04
tags:
categories: linux
---
# 安装
```shell
git clone https://github.com/likaiyuan00/openvpn-install.git
cd openvpn-install && bash openvpn-install.sh
#systemctl start openvpn@client.service 启动的账号密码  auth-user-pass 控制客户端密码验证
echo "test test@123" >  /etc/openvpn/userfile.sh
```

# 配置文件字段解读
## server端
```config
在#openvpn服务端的监听地址
local 0.0.0.0
#openvpn服务端的监听端口（默认1194）
port 1115
#使用的协议，tcp/udp
proto tcp
#使用三层路由ip隧道（tun），还是二层以太网隧道（tap），一般使用tun
dev tun
#ca证书、服务端证书、服务端秘钥和秘钥交换文件
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem
#vpn服务端为自己和客户端分配的ip地址池。
#服务端自己获取网段的第一个地址（此处是10.8.0.1），后为客户端分配其他的可用地址。以后客户端就可以和10.8.0.1进行通信。
注意：以下网段地址不要和已有网段冲突或重复
server 10.8.0.0  255.255.255.0
#使用一个文件记录已分配虚拟ip的客户端和虚拟ip的对应关系。以后openvpn重启时，将可以按照此文件继续为对应的客户端分配此前相同的ip（自动续借ip）
ifconfig-pool-persist ipp.txt
#使用tap模式的时候考虑此选项
server-bridge XXXXXX
#vpn服务端向客户端推送vpn服务端内网网段的路由配置，以便让客户端能够找到服务端的内网。多条路由写多个push指令
push "route 10.0.10.0  255.255.255.0"
push "route 192.168.10.0 255.255.255.0"  #允许客户端访问的内网网段
#让vpn客户端之间可以通信。默认情况客户端只能服务端进行通信
#默认此项是注释的，客户端之间不能相互通信
client-to-client
#允许多个客户端使用同一个vpn账号连接服务端
#默认是注释的，不支持多个客户端登录一个账号
duplicate-cn
#每10秒ping一次，120秒后没收到ping就说明对方挂了
keepalive 10 120
#加强认证方式，防攻击。如果配置文件中启用此项（默认是启用的），需要执行openvpn --genkey --secret ta.key，并把ta.key放到/etc/openvpn/server/目录，服务端第二个参数为0；同时客户端也要有此文件，且client.conf中此指令的第二个参数需要为1
tls-auth /etc/openvpn/server/ta.key 0
#选择一个密码。如果在服务器上使用了cipher选项，那么也必须在这里指定它。注意，v2.4客户端/服务端将在tls模式下自动协商AES-256-GCM
cipher AES-256-CBC
#openvpn 2.4版本的vpn才能设置此选项。表示服务端启用lz4的压缩功能 ，传输数据给客户端时会压缩数据包。
Push后在客户端也配置启用lz4的压缩功能，向服务端发数据时也会压缩。如果是2.4版本以下的老版本，则使用用comp-lzo指令
compress lz4-v2
push "compress lz4-v2"
#启用lzo数据压缩格式，此指令用于低于2.4版本的老版本，且如果服务端配置了该指令，客户端也必须要配置
comp-lzo
#并发客户端的连接数
max-clients 1000
#通过ping得知超时时，当重启vpn后将使用同一个秘钥文件以及保持tun连接状态
persist-key
persist-tun
#在文件中输出当前的连接信息，每分钟截断并重写一次该文件
status openvpn-status.log
#log指令表示每次启动vpn时覆盖式记录到指定日志文件中
#log-append则表示每次启动vpn时追加式的记录到指定日志中
#但两者只能选其一，或者不选时记录到rsyslog中
log  /var/log/openvpn.log
log-append  /var/log/openvpn.log
#日志记录的详细级别
verb 3
#当服务器重新启动时，通知客户端，以便它可以自动重新连接。仅在UDP协议是可用
explicit-exit-notify 1
#沉默的重复信息。最多20条相同消息类别的连续消息将输出到日志
mute 20
```
## client
```
#标识这是个客户端
client
#使用的协议，tcp/udp，服务端是什么客户端就是什么
proto tcp
#使用三层路由ip隧道（tun），还是二层以太网隧道（tap），服务端是什么客户端就是什么
dev tun
#服务端的地址和端口
remote 10.0.0.190 1194
#一直尝试解析OpenVPN服务器的主机名
resolv-retry infinite
#大多数客户机不需要绑定到特定的本地端口号
nobind
#初始化后的降级特权(仅非windows)
user nobody
group nobody
#尝试在重新启动时保留某些状态
persist-key
persist-tun
#ca证书、客户端证书、客户端密钥
#如果它们和client.conf或client.ovpn在同一个目录下则可以不写绝对路径，否则需要写绝对路径调用
ca ca.crt
cert client.crt
key client.key
#通过检查certicate是否具有正确的密钥使用设置来验证服务器证书。
remote-cert-tls server
#加强认证方式，防攻击。服务端有配置，则客户端必须有
tls-auth ta.key 1
#选择一个密码。如果在服务器上使用了cipher选项，那么您也必须在这里指定它。注意，v2.4客户端/服务器将在TLS模式下自动协商AES-256-GCM。
cipher AES-256-CBC
# 服务端用的什么，客户端就用的什么
#表示客户端启用lz4的压缩功能，传输数据给客户端时会压缩数据包
comp-lzo
# 日志级别
verb 3
#沉默的重复信息。最多20条相同消息类别的连续消息将输出到日志
mute 20
```

# 如何直连openvpn服务端其他局域网服务器
> 客户端（10.8.0.10） <br>
ping (服务端)172.16.1.7 正常 <br>
ping (服务端其他内网机器)172.16.1.8失败
>>1. 第一种方法 配置路由
route add -net 10.8.0.0 netmask 255.255.255.0 gw 172.16.1.7<br>
10.8.0.0  客户端IP<br>
172.16.1.7 openvpn 服务端IP
***
>>2. 第二种方法使用snat转发 <br>
iptables -t nat -A POSTROUTING -d 10.8.0.0/24 -o eth0 -j MASQUERADE<br>
iptables -A FORWARD -s 10.8.0.0 -j ACCEPT


# 额外
服务端
route 192.168.0.0 255.255.0.0   指令作用是在服务端加一条路由，网关是客户端ip
![alt text](image.png)

服务端只能ping通客户端的tun0的ip，内网ip不行，即使加了路由也不行
![alt text](image-1.png)

客户端
push "route 192.168.10.0 255.255.255.0"作用是在客户端多加一条路由。网关是服务端的tun0IP（也就是server 指令配置分配的地址池）
![alt text](image-2.png)
![alt text](image-3.png)

