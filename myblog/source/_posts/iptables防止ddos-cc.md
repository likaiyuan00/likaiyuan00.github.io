---
title: iptables防止ddos(cc)
date: 2025-04-21 19:17:48
tags:
categories: linux
---
> 基本上发行版都是自带的，轻量级，不需要额外下载
Fail2Ban也可以但是需要额外下载

# 如何配置使用
```shell
iptables -I INPUT -p tcp --dport 80 -m state --state NEW -m recent --set

参数    作用
-I INPUT    将规则插入到 INPUT 链的最前面
-p tcp --dport 80    匹配目标端口为 80 的 TCP 流量
-m state --state NEW    仅匹配 新建连接（如 TCP 的 SYN 包）
-m recent --set    将来源 IP 记录到 recent 模块的默认列表（/proc/net/xt_recent/DEFAULT）

iptables -I INPUT -p tcp --dport 80 -m state --state NEW -m recent --update --seconds 60 --hitcount 100 -j DROP

参数    作用
-m recent --update --seconds 60 --hitcount 100    检查 IP 在 60 秒内是否发起超过 100 次新连接
-j DROP    若超限，直接丢弃数据包
```

## 效果图，到指定次数自动丢弃数据包，端口不通，到达指定时间自动恢复

![alt text](image.png)
![alt text](image-1.png)



## 经过测试 --hitcount 大于20 会报错
![alt text](image-2.png)

### 解决办法 
```shell
echo options xt_recent ip_pkt_list_tot=200 > /etc/modprobe.d/xt.conf

modprobe -r xt_recent && modprobe xt_recent 重新加载

查看 lsmod |grep xt  ；cat /sys/module/xt_recent/parameters/ip_pkt_list_tot 对应 xt.conf
```
# 额外补充

若其他规则也使用 recent 默认列表，可能导致误判，可以通过--name 指定名称分类

iptables -I INPUT -p tcp --dport 80 -m state --state NEW -m recent --set --name HTTP_CC

iptables -I INPUT -p tcp --dport 80 -m state --state NEW -m recent --update --seconds 60 --hitcount 200 --name HTTP_CC -j DROP

则 /proc/net/xt_recent/HTTP_CC 叫 HTTP_CC

