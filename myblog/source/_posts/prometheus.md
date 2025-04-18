---
title: prometheus
date: 2025-04-18 09:56:59
tags: prometheus
categories: prometheus
---
https://github.com/likaiyuan00/k8s-prometheus.git

# k8s-prometheus
部署kubernetes_sd_configs
配置文件只采集了
> 1 prometheus*  prometheus-server<br> 
> 2 container*   kubelet 的10250端口  /metrics/cadvisor<br>
> 3 node*    node_exporter<br>
> 4 apiserver*  apiserver 6443 端口 /metrics<br>
> 5 kube*  kube-state-metrics组件 8080端口 /metrics<br>
> 6 coredns*  kubernetes-pods 自动发现 pod需要配置 prometheus.io/scrape: "true" 不然抓取不到 默认flase<br>
> prometheus.io/path: "/metrics"   # 指标路径（默认 /metrics 可不写）<br>
> 7 kubelet*  apiserver代理端点 /api/v1/nodes/\<node-name\>/proxy/metrics
其他有需要的可以自行配置


导入镜像，执行yml文件即可

## prometheus效果图
![alt text](image-2.png)


## grafana效果图
![alt text](image-3.png)
![alt text](image-4.png)
![alt text](image-5.png)




## kubelet 组件
 kubelet 三个指标 /metrics/probes（探针） /metrics/cadvisor（pod） /metrics（node）

对应apiserver的 /api/v1/nodes/${node-name}/proxy/${url};一般为了减少apiserver的负载不建议使用这种方式 **

直接访问会报401没有权限
![alt text](image-6.png)


需要先获取token，上面文件执行完会有一个prometheus用户
![alt text](image-7.png)



pod内token路径为 /var/run/secrets/kubernetes.io/serviceaccount/token

通过token再去访问发现就正常了

```
/metrics
curl -k -sS  -H "Authorization: Bearer $TOKEN"  https://127.0.0.1:6443/api/v1/nodes/master/proxy/metrics

curl -k -sS  -H "Authorization: Bearer $TOKEN"  https://127.0.0.1:10250/metrics
```
![alt text](image-8.png)



对应kubelet*开头
![alt text](image-9.png)


```
/metrics/probes（探针）
curl -k -sS  -H "Authorization: Bearer $TOKEN"  https://127.0.0.1:6443/api/v1/nodes/master/proxy/metrics/probes

curl -k -sS  -H "Authorization: Bearer $TOKEN"  https://127.0.0.1:10250/metrics/probes
```
![alt text](image-10.png)

```
/metrics/cadvisor（pod）
curl -k -sS  -H "Authorization: Bearer $TOKEN"  https://127.0.0.1:6443/api/v1/nodes/master/proxy/metrics/cadvisor

curl -k -sS  -H "Authorization: Bearer $TOKEN"  https://127.0.0.1:10250/metrics/cadvisor
```
![alt text](image-11.png)




对应container*开头，容器指标
![alt text](image-12.png)


## node_exporter  
端口暴露到节点了就不需要token了
![alt text](image-13.png)


node*开头，节点指标
![alt text](image-14.png)



## kube-state-metrics
集群应用状态监控比较重要的一个需要单独安装
使用containerPort: 8080 暴露到节点了不需要token
![alt text](image-15.png)




kube*开头
![alt text](image-16.png)


## apiserver 
主要是监控apiserver的qps,查询成功率失败率等信息
![alt text](image-17.png)


apiserver*开头
![alt text](image-18.png)


## kubernetes-pods 自动发现
如果元数据内设置true，该pod才可以被抓取，默认false
![alt text](image-19.png)



以coredns为例
![alt text](image-20.png)
![alt text](image-21.png)





以coredns*开头
![alt text](image-22.png)


这个自动发现还可以配置自身业务的监控，只有保证开启抓取，和符合prometheus抓取规范就可以，如果开启了prometheus.io/scrape 但是pod并没有提供数据指标的能力就会直接报错，如图404
![alt text](image-23.png)


比如现在我想加一个grafana的数据，只需要添加对应元数据就可以了
![alt text](image-24.png)


prometheus就自动发现了pod的ip
![alt text](image-1.png)


grafana*开头
![alt text](image.png)



