---
title: prometheus进阶
date: 2025-08-22 15:25:19
tags:
categories: prometheus
---
# deploy
```yml
#!/bin/bash

# 创建目录结构
mkdir -p monitoring/prometheus
cd monitoring

# 生成docker-compose.yml,node和process必须使用host宿主机网络，不然很多指标只能采集到容器里面的信息不准确
cat > docker-compose.yml << EOF
version: '3.8'

networks:
  monitoring:
    driver: bridge

services:
  prometheus:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/prometheus:v2.40.7
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prom_data:/prometheus
    networks:
      - monitoring
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle' #curl -X POST http://localhost:9090/-/reload

  node-exporter:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/nodeexporter:v1.9.1
    container_name: node-exporter
    restart: unless-stopped
    command:
      - '--path.rootfs=/host'
      - '--web.listen-address=:9400'
   # networks:  # 加入监控网络
    #  - monitoring
    network_mode: host  # 使用host网络
    pid: host
    volumes:
      - /:/host:ro,rslave

  blackbox-exporter:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/nodeexporter:blackbox-exporterv0.27.0
    container_name: blackbox-exporter
    restart: unless-stopped
    ports:
      - "9115:9115"
    networks:
      - monitoring
    volumes:
      - ./blackbox.yml:/etc/blackbox_exporter/config.yml
    command:
      - '--config.file=/etc/blackbox_exporter/config.yml'

  process-exporter:
    image: registry.cn-hangzhou.aliyuncs.com/lky-deploy/nodeexporter:process-exporter-v0.8.7
    container_name: process-exporter
    restart: unless-stopped
    network_mode: host  # 使用host网络
   # networks:
   #   - monitoring  # 使用统一网络
    ports:
      - "9256:9256"  # 添加端口映射
    volumes:
      - /proc:/host/proc:ro
      - ./process-exporter.yml:/config.yml
    command:
      - '-config.path=/config.yml'
      - '-procfs=/host/proc'

volumes:
  prom_data:
EOF

# 生成Prometheus配置
cat > prometheus/prometheus.yml <<EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9400'] #host模式需要换成宿主机ip
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://qq.com
        - https://google.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'process-exporter'
    static_configs:
      - targets: ['process-exporter:9256'] #host模式需要换成宿主机ip
EOF

# 生成Blackbox配置
cat > blackbox.yml <<EOF
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      valid_status_codes: [200]
      method: GET
      preferred_ip_protocol: "ip4"
EOF

# 生成Process Exporter配置
cat > process-exporter.yml <<EOF
process_names:
  - name: "{{.Comm}}"
    cmdline:
    - '.+'  # 正则表达式（匹配任何非空内容）
  #- name: "{{.Matches}}"
 #   cmdline:
 #     - 'nginx'  # 只监控 nginx 进程
 # - name: "{{.Matches}}"
  #  cmdline:
  #    - 'docker'  # 只监控 docker 进程
EOF

# 启动服务
docker compose up -d

echo -e "\n\033[32m部署完成！以下是访问信息：\033[0m"
echo "Prometheus:     http://localhost:9090"
echo "Node Exporter:  http://localhost:9100/metrics"
echo "Blackbox:       http://localhost:9115/metrics"
echo "Process-Exporter: http://localhost:9256/metrics"
echo -e "\n请修改以下配置文件后重启服务："
echo "- blackbox.yml 中的监控目标"
echo "- process-exporter.yml 中的进程过滤规则"

```


# 如果节点较多prometheus对所有指标的采集会对负载和磁盘占用较多，可以通过relabel drop不需要的指标，减轻负担
```yaml
#relabel_configs	抓取前，针对target
#metric_relabel_configs 抓取后，针对指标名称
scrape_configs:
  - job_name: 'node-drop' 
    static_configs:
      - targets: ['localhost:9100']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: '^(node_cpu_seconds_total|node_memory_.*|node_disk_.*|node_network_.*)$'
        action: keep
  - job_name: 'node'
    static_configs:
      - targets: ['node1:9100', 'node2:9100','master:9100']
    relabel_configs:
      # 根据目标地址动态添加 environment 标签
      - source_labels: [__address__]
        regex: 'node1:9100'
        replacement: 'prod'
        target_label: environment
      - source_labels: [__address__]
        regex: 'node2:9100'
        replacement: 'staging'
        target_label: environment
      # 只保留主机名包含 "node" 的目标
      - source_labels: [__address__]
        regex: 'node[0-9]+:9100'  # 正则匹配 node1, node2 等
        action: keep
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: "nginx|api-server"  # 只抓取带有 app=nginx 或 app=api-server 标签的 Pod
        action: keep


# 仅启用 cpu 和 meminfo 收集器;不好用
node_exporter \
  --collector.cpu \
  --collector.meminfo \
  --no-collector.diskstats \
  --no-collector.netdev \
  --no-collector.filesystem \
  # 禁用其他所有收集器...
```

# file_sd
可以基于文件动态更新 prometheus 的监控节点
```json
#文件类型/etc/prometheus/targets/nodes.json
[
  {
    "targets": ["192.168.1.10:9100"],  # 监控目标地址（IP:Port）
    "labels": {                        # 自定义标签（可选）
      "env": "prod",
      "role": "web-server"
    }
  },
  {
    "targets": ["192.168.1.11:9100","192.168.3.11:9100"],
    "labels": {
      "env": "staging",
      "role": "db-server"
    }
  }
]
#prometheus配置
scrape_configs:
  - job_name: "node-exporter"            # 任务名称
    file_sd_configs:                     # 启用 file_sd
      - files:
          - "/etc/prometheus/targets/*.json"  # 目标文件路径（支持通配符）
          - "/etc/prometheus/targets/mysql-exporters/*.json" # MySQL 监控
        refresh_interval: 5m             # 重新加载间隔（默认 5m）

```
# node exporter textfile
```shell
#!/bin/bash

DURATION=15         # 默认抓包时长（建议比 cron 间隔稍短）
INTERFACE="eth0"
OUTPUT_FILE="/tmp/traffic.pcap"
METRICS_FILE="/etc/node-exporter/textfile-collector/network_traffic.prom"  # Node Exporter 收集目录

# 安装依赖（如未安装）
if ! command -v tcpdump &>/dev/null || ! command -v tshark &>/dev/null; then
    echo "安装依赖: tcpdump 和 tshark..."
    sudo apt-get update && sudo apt-get install -y tcpdump tshark
fi

# 捕获流量
sudo timeout $DURATION tcpdump -i $INTERFACE -w $OUTPUT_FILE >/dev/null 2>&1

# 生成 Prometheus 格式的指标
sudo tshark -r $OUTPUT_FILE -T fields -e ip.src -e ip.dst -e frame.len 2>/dev/null \
  | awk '
    BEGIN {
        total_bytes = 0
        delete bytes  # 清空数组
    }
    {
        bytes[$1] += $3;  # 源IP统计
        bytes[$2] += $3;  # 目的IP统计
        total_bytes += $3
    }
    END {
        # 输出总流量指标
        print "network_traffic_total_bytes " total_bytes

        # 输出每个IP的流量指标
        for (ip in bytes) {
            if (ip != "") {  # 过滤空值
                printf "network_traffic_bytes{ip=\"%s\"} %d\n", ip, bytes[ip]
            }
        }
    }' > "$METRICS_FILE.$$"  # 先写入临时文件

# 原子操作替换文件（避免读取半成品）
sudo mv "$METRICS_FILE.$$" "$METRICS_FILE"

# 清理
sudo rm -f "$OUTPUT_FILE"


./node_exporter  --web.listen-address=":900" --collector.textfile.directory=/etc/node-exporter/textfile-collector/
```







# blackbox_exporter
```yml
#blackbox.yml配置，prober类型可以自定义http,tcp,icmp,dns
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      valid_status_codes: [200]
      method: GET
      preferred_ip_protocol: "ip4"

  ssh_banner_check:  # 自定义模块名
    prober: tcp
    timeout: 10s
    tcp:
      query_response:
        - expect: "^SSH-2.0-OpenSSH"
          send: "SSH-2.0-blackbox-ssh-check"
      preferred_ip_protocol: "ip4"


# prometheus集成
scrape_configs:
 - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://qq.com
        - https://google.com
    relabel_configs: &common_relabel
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'blackbox-ssh'
    metrics_path: /probe
    params:
      module: [ssh_banner_check]  # 对应TCP模块
    static_configs:
      - targets:
        - '127.0.0.1:22'
        - '10.0.1.122:22'
    relabel_configs: *common_relabel

#relabel_configs配置解析
Prometheus 抓取任务生成
│
├─ 原始目标: example.com:80
│  │
│  ├─ relabel 规则1: 将地址赋值给 __param_target → ?target=example.com:80
│  ├─ relabel 规则2: 用 __param_target 标记 instance → instance="example.com:80"
│  └─ relabel 规则3: 重写地址 → 实际请求发送到 Blackbox Exporter
│     │
│     └─ Blackbox Exporter 收到请求，解析参数后探测 example.com:80
│
└─ 原始目标: google.com:443
   └─ 同理生成请求 http://192.168.100.100:9115/probe?target=google.com:443&module=http_2xx


#告警ing
probe_success{job="blackbox-http"} == 0
#状态码
probe_http_status_code{job="blackbox-http"} < 200 or probe_http_status_code{job="blackbox-http"} >= 300
#证书过期时间
probe_ssl_earliest_cert_expiry{job="blackbox-http"} - time() < 86400 * 30  # 30天
#tcp端口是否通
probe_success{job="blackbox-tcp"} == 0
#响应时长
probe_duration_seconds{job="blackbox-http"} > 1
#解析时长
probe_dns_lookup_time_seconds
```





# process-exporter
```yml
{{.Comm}} 包含原始可执行文件的基本名称，即 /proc/<pid>/stat 中的第 2 个字段，并截取前15个字符
{{.ExeBase}} 包含可执行文件的基本名称  
{{.ExeFull}} 包含可执行文件的完全限定路径  
{{.Username}} 包含有效用户的用户名  
{{.Matches}} map 包含应用 cmdline 正则表达式产生的所有匹配项
{{.PID}} 包含进程的 PID。请注意，使用 PID 意味着该组将仅包含一个进程
{{.StartTime}} 包含进程的开始时间。这与 PID 结合使用时非常有用，因为 PID 会随着时间的推移而被重用。
{{.Cgroups}} 包含（如果支持）进程的 cgroups （/proc/self/cgroup）。这对于识别进程属于哪个容器特别有用

#process-exporter.yml
#常用的就comm和exefull,matches
process_names:
  #- name: "{{.ExeFull}}" 
  #  cmdline:
  #  - '.+'  # 正则表达式（匹配任何非空内容）不常用太多了影响资源消耗
  - name: "{{.Comm}}" #groupname="docker"
    cmdline:
    - 'docker*' 
  - name: "{{.Matches}}" #groupname="map[:nginx]"
    cmdline:
    - 'nginx*' 
  - name: "{{.ExeFull}}"  #groupname="/usr/sbin/mysqld
    cmdline:
    - 'mysql*'
                




#告警...
namedprocess_namegroup_num_procs{groupname!~".*process-exporter.*"} == 0
namedprocess_namegroup_states{state="Z"} > 0
namedprocess_namegroup_num_procs{groupname="nginx"} == 0
#cpu百分比
100 * rate(namedprocess_namegroup_cpu_seconds_total{groupname="java"}[5m])
100 * rate(namedprocess_namegroup_cpu_seconds_total{}[5m]) > 50

#内存 mb
namedprocess_namegroup_memory_bytes{groupname="java"} / 1024^2
除于
#节点内存 就是占用内存百分比
node_memory_MemTotal_bytes / 1024 / 1024 

# 读速率 MB/S
rate(namedprocess_namegroup_read_bytes_total{groupname="mysql"}[5m]) / 1024^2

# 写速率
rate(namedprocess_namegroup_write_bytes_total{groupname="mysql"}[5m]) / 1024^2


```

# prometheus联邦
* 数量较多的情况下从多个下级 Prometheus 实例中提取特定指标，汇总到中心 Prometheus
```yml

                Central Prometheus
                        ↑
        从多个下级拉取聚合后的指标
                        |
        +---------------+---------------+
        |               |               |
   Region A       Region B       Region C
   Prometheus    Prometheus    Prometheus


scrape_configs:
  - job_name: 'federate-regions'        # 任务名称
    scrape_interval: 1m                # 建议比下级采集间隔长
    honor_labels: true                 # 保留下级标签（避免覆盖）
    metrics_path: '/federate'          # 联邦接口路径
    params:
      'match[]':
        - '{job="api-server"}'         # 拉取下级的指定 job 指标
        - '{__name__=~"job:.*"}'
        - 'up{instance=~".+"}'         # 拉取所有下级实例的 up 状态
    static_configs:
      - targets:
          - 'prometheus-region-a:9090' # 下级 Prometheus 地址
          - 'prometheus-region-b:9090'
          - 'prometheus-region-c:9090'

match[] 过滤条件
精确匹配：'{job="mysql"}' 拉取所有 job=mysql 的指标。
正则匹配：'__name__=~"http_request_.+"' 拉取以 http_request_ 开头的指标。
组合条件：'{env="prod", app=~"web|api"}' 拉取 prod 环境下 web 或 api 应用的指标。
```
