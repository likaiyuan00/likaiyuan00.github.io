---
title: alertmanager
date: 2025-04-27 16:06:39
tags:
categories: prometheus
---
# 安装
```shell
curl -SL https://github.com/docker/compose/releases/download/v2.30.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
#将可执行权限赋予安装目标路径中的独立二进制文件
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

version: '3'
services:
  alertmanager:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/alertmanager:v0.28.1
    ports:
      - "9093:9093"
      - "9094:9094"
    volumes:
      - ./config:/etc/alertmanager
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--cluster.advertise-address=alertmanager:9094'
    networks:
      - monitoring-net

volumes:
  alertmanager_data:

networks:
  monitoring-net:
    driver: bridge

```
# 配置文件解读

```
global: # 即为全局设置,在Alertmanager配置文件中,只要全局设置配置了的选项,全部为公共设置,可以让其他设置继承,作为默认值,可以子参数中覆盖其设置。
  resolve_timeout: 1m # 用于设置处理超时时间,也是生命警报状态为解决的时间,这个时间会直接影响到警报恢复的通知时间,需要自行结合实际生产场景来设置主机的恢复时间,默认是5分钟。
  # 整合邮件
  smtp_smarthost: 'smtp.qq.com:465' # 邮箱smtp服务器
  smtp_from: '1451578387@qq.com' # 发件用的邮箱地址
  smtp_auth_username: '1451578387@qq.com' # 发件人账号
  smtp_auth_password: 'dkuuifhdskaduasdsb' # 发件人邮箱密码
  smtp_require_tls: false # 不进行tls验证
route: # 路由分组
  group_by: ['alertname'] # 报警分组
  group_wait: 10s # 组内等待时间,同一分组内收到第一个告警等待多久开始发送,目标是为了同组消息同时发送,不占用告警信息,默认30s。
  group_interval: 10s # 当组内已经发送过一个告警,组内若有新增告警需要等待的时间,默认为5m,这条要确定组内信息是影响同一业务才能设置,若分组不合理,可能导致告警延迟,造成影响。
  repeat_interval: 1h # 告警已经发送,且无新增告警,若重复告警需要间隔多久,默认4h,属于重复告警,时间间隔应根据告警的严重程度来设置。
  receiver: 'webhook' # 告警的接收者,需要和 receivers[n].name 的值一致。
  # 上面所有的属性都由所有子路由继承,并且可以在每个子路由上进行覆盖。
  # 当报警信息中标签匹配到team:node时会使用email发送报警,否则使用webhook。
 templates:
- '/etc/alertmanager/config/*.tmpl'
# route根路由,该模块用于该根路由下的节点及子路由routes的定义,子树节点如果不对相关配置进行配置,则默认会从父路由树继承该配置选项。每一条告警都要进入route,即要求配置选项group_by的值能够匹配到每一条告警的至少一个labelkey(即通过POST请求向altermanager服务接口所发送告警的labels项所携带的<labelname>),告警进入到route后,将会根据子路由routes节点中的配置项match_re或者match来确定能进入该子路由节点的告警(由在match_re或者match下配置的labelkey:labelvalue是否为告警labels的子集决定,是的话则会进入该子路由节点,否则不能接收进入该子路由节点)。
route:
  # 例如所有labelkey:labelvalue含cluster=A及altertname=LatencyHigh的labelkey的告警都会被归入单一组中
  group_by: ['job', 'altername', 'cluster', 'service','severity']
  # 若一组新的告警产生,则会等group_wait后再发送通知,该功能主要用于当告警在很短时间内接连产生时,在group_wait内合并为单一的告警后再发送
  group_wait: 30s
  # 再次告警时间间隔
  group_interval: 5m
  # 如果一条告警通知已成功发送,且在间隔repeat_interval后,该告警仍然未被设置为resolved,则会再次发送该告警通知
  repeat_interval: 12h
  # 默认告警通知接收者,凡未被匹配进入各子路由节点的告警均被发送到此接收者
  receiver: 'wechat'
  # 上述route的配置会被传递给子路由节点,子路由节点进行重新配置才会被覆盖
  # 子路由树
  routes:
  # 该配置选项使用正则表达式来匹配告警的labels,以确定能否进入该子路由树
  # match_re和match均用于匹配labelkey为service,labelvalue分别为指定值的告警,被匹配到的告警会将通知发送到对应的receiver
  - match_re:
      service: ^(foo1|foo2|baz)$
    receiver: 'wechat'
    # 在带有service标签的告警同时有severity标签时,他可以有自己的子路由,同时具有severity != critical的告警则被发送给接收者team-ops-mails,对severity == critical的告警则被发送到对应的接收者即team-ops-pager
    routes:
    - match:
        severity: critical
      receiver: 'wechat'
  # 比如关于数据库服务的告警,如果子路由没有匹配到相应的owner标签,则都默认由team-DB-pager接收
  - match:
      service: database
    receiver: 'wechat'
  # 我们也可以先根据标签service:database将数据库服务告警过滤出来,然后进一步将所有同时带labelkey为database
  - match:
      severity: critical
    receiver: 'wechat'
# 抑制规则,当出现critical(关键的)告警时忽略warning。
# 下面的这段配置是指如果出现标签为severity=critical的告警,则抑制severity=warning的告警
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  # 如果警报名称相同,则应用抑制。
  # alertname、cluster和service对应的标签值需要相等
  equal: ['alertname', 'cluster', 'service']
# 收件人配置
receivers:
- name: 'team-ops-mails'
  email_configs:
  - to: 'dukuan@xxx.com'
- name: 'team-X-pager'
  email_configs:
  - to: 'team-X+alerts-critical@example.org'
  pagerduty_configs:
  - service_key: <team-X-key>
- name: 'team-Y-mails'
  email_configs:
  - to: 'team-Y+alerts@example.org'
- name: 'webhook'
  webhook_configs:
  - url: http://127.0.0.1:8060/dingtalk/webhook1/send
    send_resolved: true
```
## 分组和路由
>1. **路由**<br>
match（精确匹配）match_re（正则表达式匹配）
>每一个告警都会从配置文件中顶级的route进入路由树，需要注意的是顶级的route必须匹配所有告警(即不能有任何的匹配设置match和match_re)，每一个路由都可以定义自己的接受人以及匹配规则。默认情况下，告警进入到顶级route后会遍历所有的子节点，直到找到最深的匹配route，并将告警发送到该route定义的receiver中。但如果route中设置continue的值为false，那么告警在匹配到第一个子节点之后就直接停止。如果continue为true，报警则会继续进行后续子节点的匹配。如果当前告警匹配不到任何的子节点，那该告警将会基于当前路由节点的接收器配置方式进行处理
>2. **分组**<br>
>告警通知进行分组，将多条告警合合并为一个通知。这里我们可以使用group_by来定义分组规则。基于告警中包含的标签，如果满足group_by中定义标签名称，那么这些告警将会合并为一个通知发送给接收器。
有的时候为了能够一次性收集和发送更多的相关信息时，可以通过group_wait参数设置等待时间，如果在等待时间内当前group接收到了新的告警，这些告警将会合并为一个通知向receiver发送

```
route:
  group_by: ['alertname','team']   #在这里添加team匹配的标签
  group_wait: 5s
  group_interval: 5s
  repeat_interval: 5m
  # 默认发给"sre_system"组用户
  receiver: 'sre_system'
  continue: false
  # 配置子路由
  routes:
    - receiver: 'sre_dba'
      match_re:
        job: test
      # 建议将continue的值设置为true，表示当前的条件是否匹配，都将继续向下匹配规则
      # 这样做的目的是将消息发给最后的系统组(sre_system)
      continue: true
==================================================================
#rule.yml
- name: grafana
  rules:
  - alert: node           #这个相当于alertname的值,与之前匹配的相同
    expr: up{job="grafana"} == 0
    for: 10s                  
    labels:                   
      severity: 1 
      job: test  # 对应上面的 match_re
      team: grafana        #这里标签设置不同的一会用
    annotations:              
      summary: "{{ \$labels.instance }} 已停止运行超过 15s"
      description: hello world

 
alertname 等于 node 如果相同报警会一起发送
team 等于 grafana 
```
## 抑制规则
```
inhibit_rules:
  - source_match:
      severity: '告警'
    target_match:
      severity: '提示'
    #equal: ['type','test'] 要求 type 和 test签均相同
     equal: ['type']type的值必须一样
当匹配到 告警 时就会抑制提示的告警通知并检查他们是否来自于同个
ssl（即ssl标签的值相同抑制才会生效）
当子路由匹配到不同的 severity 时就会将消息发往不同的 receiver，当子路由无法匹配到时，消息会默认发往根路由的 receiver，因此，无论是否匹配到子路由规则，消息都会发往根路由的 receiver
对应报警规则配置为
groups:
- name: node-alerts
  rules:
  - alert: HighNodeCPU
    expr: (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100 > 80
    for: 5m
    labels:
      severity: "告警"
      type: ssl
    annotations:
      summary: "高节点CPU使用率 ({{ $labels.instance }})"
      description: "节点 {{ $labels.instance }} CPU 使用率超过 80% 已持续 5 分钟"

- name: cluster-alerts
  rules:
  - alert: ClusterWideCPUProblem
    expr: |
      sum( (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100 > 80 )
      /
      count( (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) )
      * 100 > 50
    for: 10m
    labels:
      severity: "提示"
      type: ssl
    annotations:
      summary: "集群级CPU问题"
      description: "超过 50% 的节点持续高CPU使用率达 10 分钟"
```

# alertmanager集成三方告警
>原生alertmanager只有邮件和webhook告警；Alertmanager 的原生 Webhook 告警是一种通过 HTTP POST 请求将告警信息发送到自定义接口（Webhook 接收端）的机制；所以要对接需要开发者自行开发，这里推荐两个现成的工具来对接

## PrometheusAlert使用
>Prometheus Alert 是开源的运维告警中心消息转发系统，支持主流的监控系统 Prometheus，日志系统 Graylog 和数据可视化系统 Grafana 发出的预警消息。通知渠道支持钉钉、微信、华为云短信、腾讯云短信、腾讯云电话、阿里云短信、阿里云电话等等
```shell
wget https://github.com/feiyu563/PrometheusAlert/releases/download/v4.8.1/linux.zip
chmod +x PrometheusAlert
启动 nohup ./PrometheusAlert & 后台运行

#alertmanager.yml配置集成PrometheusAlert；格式可以登录PrometheusAlert查看
receivers:
- name: 'web.hook.prometheusalert'
  webhook_configs:
  - url: 'http://192.168.197.142:8080/prometheusalert?type=wx&tpl=prometheus-wx&wxurl=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=53fdb356-4446-42e5-b8bd-f7da63bcfe76'
```
![alt text](image.png)


## prometheus-webhook-dingtalk使用
>Prometheus 的Alertmanager自身不支持钉钉报警，需要通过插件的方式来达到报警条件
### 安装
```shell
wget https://github.com/timonwong/prometheus-webhook-dingtalk/releases/download/v2.1.0/prometheus-webhook-dingtalk-2.1.0.linux-amd64.tar.gz

tar zxf prometheus-webhook-dingtalk-2.1.0.linux-amd64.tar.gz 
mv prometheus-webhook-dingtalk-2.1.0.linux-amd64 /usr/local/prometheus-webhook-dingtalk
cat > /usr/lib/systemd/system/webhook-dingtalk.service << EOF
[Unit]
Description=prometheus-webhook-dingtalk
Documentation=https://github.com/timonwong/prometheus-webhook-dingtalk
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/local/prometheus-webhook-dingtalk/prometheus-webhook-dingtalk  --config.file=/usr/local/prometheus-webhook-dingtalk/config.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
#集成模版/usr/local/prometheus-webhook-dingtalk/config.yml
templates:
    - /usr/local/prometheus/webhook-dingtalk/template.tmpl
targets:
  webhook1:
    url: https://oapi.dingtalk.com/robot/send?access_token=9ac4354ab7c8

#alertmanager配置发送给dingtalk插件
receivers:
  - name: 'email'
    email_configs:
      - to: 'xxxxx@163.com' #指定发送给谁
  - name: 'webhook1'
    webhook_configs:
      - send_resolved: false
        url: http://localhost:8060/dingtalk/webhook1/send
```

### 报警内容模版
```config
vim /usr/local/prometheus/webhook-dingtalk/template.tmpl

{{ define "__subject" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
{{ end }}
 
 
{{ define "__alert_list" }}{{ range . }}
---
{{ if .Labels.owner }}@{{ .Labels.owner }}{{ end }}
 
**告警主题**: {{ .Annotations.summary }}

**告警类型**: {{ .Labels.alertname }}
 
**告警级别**: {{ .Labels.severity }} 
 
**告警主机**: {{ .Labels.instance }} 
 
**告警信息**: {{ index .Annotations "description" }}
 
**告警时间**: {{ dateInZone "2006.01.02 15:04:05" (.StartsAt) "Asia/Shanghai" }}
{{ end }}{{ end }}
 
{{ define "__resolved_list" }}{{ range . }}
---
{{ if .Labels.owner }}@{{ .Labels.owner }}{{ end }}

**告警主题**: {{ .Annotations.summary }}

**告警类型**: {{ .Labels.alertname }} 
 
**告警级别**: {{ .Labels.severity }}
 
**告警主机**: {{ .Labels.instance }}
 
**告警信息**: {{ index .Annotations "description" }}
 
**告警时间**: {{ dateInZone "2006.01.02 15:04:05" (.StartsAt) "Asia/Shanghai" }}
 
**恢复时间**: {{ dateInZone "2006.01.02 15:04:05" (.EndsAt) "Asia/Shanghai" }}
{{ end }}{{ end }}
 
 
{{ define "default.title" }}
{{ template "__subject" . }}
{{ end }}
 
{{ define "default.content" }}
{{ if gt (len .Alerts.Firing) 0 }}
**====侦测到{{ .Alerts.Firing | len  }}个故障====**
{{ template "__alert_list" .Alerts.Firing }}
---
{{ end }}
 
{{ if gt (len .Alerts.Resolved) 0 }}
**====恢复{{ .Alerts.Resolved | len  }}个故障====**
{{ template "__resolved_list" .Alerts.Resolved }}
{{ end }}
{{ end }}
 
 
{{ define "ding.link.title" }}{{ template "default.title" . }}{{ end }}
{{ define "ding.link.content" }}{{ template "default.content" . }}{{ end }}
{{ template "default.title" . }}
{{ template "default.content" . }}
```
### 报警规则示例
```config
mkdir /usr/local/prometheus/prometheus/rule
vim /usr/local/prometheus/prometheus/rule/node_exporter.yml

groups:
- name: 服务器资源监控
  rules:
  - alert: 内存使用率过高
    expr: 100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 80
    for: 3m 
    labels:
      severity: 严重告警
    annotations:
      summary: "{{ $labels.instance }} 内存使用率过高, 请尽快处理！"
      description: "{{ $labels.instance }}内存使用率超过80%,当前使用率{{ $value }}%."
          
  - alert: 服务器宕机
    expr: up == 0
    for: 1s
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} 服务器宕机, 请尽快处理!"
      description: "{{$labels.instance}} 服务器延时超过3分钟,当前状态{{ $value }}. "
 
  - alert: CPU高负荷
    expr: 100 - (avg by (instance,job)(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
    for: 5m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} CPU使用率过高,请尽快处理！"
      description: "{{$labels.instance}} CPU使用大于90%,当前使用率{{ $value }}%. "
      
  - alert: 磁盘IO性能
    expr: avg(irate(node_disk_io_time_seconds_total[1m])) by(instance,job)* 100 > 90
    for: 5m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} 流入磁盘IO使用率过高,请尽快处理！"
      description: "{{$labels.instance}} 流入磁盘IO大于90%,当前使用率{{ $value }}%."
 
 
  - alert: 网络流入
    expr: ((sum(rate (node_network_receive_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance,job)) / 100) > 102400
    for: 5m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} 流入网络带宽过高，请尽快处理！"
      description: "{{$labels.instance}} 流入网络带宽持续5分钟高于100M. RX带宽使用量{{$value}}."
 
  - alert: 网络流出
    expr: ((sum(rate (node_network_transmit_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance,job)) / 100) > 102400
    for: 5m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} 流出网络带宽过高,请尽快处理！"
      description: "{{$labels.instance}} 流出网络带宽持续5分钟高于100M. RX带宽使用量{$value}}."
  
  - alert: TCP连接数
    expr: node_netstat_Tcp_CurrEstab > 10000
    for: 2m
    labels:
      severity: 严重告警
    annotations:
      summary: " TCP_ESTABLISHED过高！"
      description: "{{$labels.instance}} TCP_ESTABLISHED大于100%,当前使用率{{ $value }}%."
 
  - alert: 磁盘容量
    expr: 100-(node_filesystem_free_bytes{fstype=~"ext4|xfs"}/node_filesystem_size_bytes {fstype=~"ext4|xfs"}*100) > 90
    for: 1m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.mountpoint}} 磁盘分区使用率过高，请尽快处理！"
      description: "{{$labels.instance}} 磁盘分区使用大于90%，当前使用率{{ $value }}%."
```
