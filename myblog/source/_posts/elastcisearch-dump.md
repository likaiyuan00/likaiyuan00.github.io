---
title: elastcisearch_dump
date: 2025-08-12 15:16:41
tags: es
categories: 中间件
---

# esdump
https://github.com/elasticsearch-dump/elasticsearch-dump
```shell
docker pull elasticdump/elasticsearch-dump
or
wget https://nodejs.org/dist/v16.18.0/node-v16.18.0-linux-x64.tar.xz
export PATH=$PATH:/root/node-v16.18.0-linux-x64/bin/
npm install elasticdump -g
#bash
PREFIX="y-logs-"
curl -s "http://localhost:9200/_cat/indices?h=index" | grep "^${PREFIX}" | while read -r INDEX_NAME; do
#INDEX_NAME="y-logs-2025"
    for TYPE in {settings,mapping,data};do
        echo "start $TYPE of index  [$INDEX_NAME]..."
        elasticdump  --input=http://localhost:9200/$INDEX_NAME     --output=http://username:passwd@dest_addr.com:9200/$INDEX_NAME     --type=$TYPE --limit 10000
    done
    echo '=============================================='
    sleep 3
done
# --limit 并发数
# --output=/data/es/${INDEX_NAME}.json 备份可以保存到文件 
# --httpAuthFile指定HTTP认证文件
# --searchBody="{\"query\":{\"term\":{\"username\": \"admin\"}}}" 过滤
```


# snapshot
```shell
#设置fs类型仓库存储路径；然后重启es
export path.repo=/usr/share/elasticsearch/data/es_snapshot
#创建仓库
curl -XPUT 'http://localhost:9200/_snapshot/y_repo' \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "fs",
    "settings": {
      "location": "/usr/share/elasticsearch/data/es_snapshot",
      "compress": true
    }
  }'
#location必须和path.repo一致
#查看
curl  -s localhost:9200/_snapshot/y_repo|jq
#删除
curl -XDELETE localhost:9200/_snapshot/y_repo
=======================================================
#创建快照
#不指定索引就是快照所有
curl -XPUT   localhost:9200/_snapshot/y_repo/snapshot_all  #?wait_for_completion=true 
#指定索引快照
curl -H "Content-Type: application/json" -XPUT localhost:9200/_snapshot/y_repo/snapshot_202508_01-02?pretty -d'
{
"indices": "y-logs-2025.08.01,y-logs-2025.08.02"
}'
#查看快照信息
curl localhost:9200/_cat/snapshots/y_repo?v
curl localhost:9200/_cat/snapshots/_all?v
#查看快照包含的索引信息
curl   -Ss localhost:9200/_snapshot/y_repo/_all|jq
curl   -Ss localhost:9200/_snapshot/y_repo/snapshot_202508_01-02|jq
curl   -Ss localhost:9200/_snapshot/y_repo/snapshot_202508_01-02/_status|jq
=======================================================
#恢复快照;目标索引名称尽量不要存在同名
curl -X POST "http://localhost:9200/_snapshot/y_repo/snapshot_all/_restore" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": "y-logs-2025.08.*",
    "rename_pattern": "y-logs-2025.08.01",
    "rename_replacement": "y_new_repo"
  }'
#indices指定恢复的索引名称;需要精确匹配如果范围较大，rename_pattern匹配不上的会直接名称不变恢复
#rename_pattern	正则表达式，匹配原始索引名称中的模式（用于捕获需要替换的部分）
#rename_replacement	替换后的新索引名称模板（使用 $1、$2 等引用正则表达式中的捕获组）
#示例json参数
'{
  "indices": "y-logs-2025.08.01"  // 仅恢复该索引，名称不变
}'
'{
  "indices": "y-logs-2025.08.01",
  "rename_pattern": "(.+)",          // 匹配整个原始名称
  "rename_replacement": "restored_$1" // 新名称：restored_y-logs-2025.08.01
}'
'{
  "indices": "y-logs*",
  "rename_pattern": "y-logs-(\\d{4})\\.(\\d{2})\\.(\\d{2})",  // 捕获年、月、日
  "rename_replacement": "archive_$1-$2-$3"  // 新名称：archive_2025-08-01
}'
'{
    "indices": "*,-.security*,-.kibana*", //使用- 过滤不迁移的索引
    "ignore_unavailable": "true"
}'
=========================================================================
#查看快照恢复情况
curl -s -X GET "http://localhost:9200/_recovery/"|jq
curl -X GET "http://localhost:9200/_cat/recovery/?v"
curl -s -X GET "http://localhost:9200/archive_2025-08-08/_recovery/"|jq
curl -s -X GET "http://localhost:9200/_cat/recovery/archive_2025-08-08?format=json"|jq
{
    "index": "archive_2025-08-08",
    "shard": "0",
    "time": "441ms", 恢复总耗时（441 毫秒）
    "type": "snapshot", 恢复类型为 快照恢复（从快照仓库恢复数据）。
    "stage": "done", 当前恢复阶段：- init（初始化）- index（复制索引文件）- translog（传输事务日志）- finalize（完成恢复）- done（已完成）
    "source_host": "n/a", 源数据节点的主机名或 IP（仅在跨节点恢复时出现，如分片迁移）
    "source_node": "n/a", 源节点的名称（Elasticsearch 集群内唯一标识）。
    "target_host": "172.26.0.2", 目标
    "target_node": "be439083dc6b",
    "repository": "y_repo",
    "snapshot": "snapshot_all",
    "files": "35", 需要恢复文件数
    "files_recovered": "35", 已经恢复
    "files_percent": "100.0%", 进度
    "files_total": "35", 总文件数
    "bytes": "3429792", 字节
    "bytes_recovered": "3429792", 已恢复字节数
    "bytes_percent": "100.0%", 进度
    "bytes_total": "3429792",
    "translog_ops": "0", 需要恢复的事务日志操作总数
    "translog_ops_recovered": "0", 
    "translog_ops_percent": "100.0%"
  }



#查看仓库级别的进度;只显示正在执行的快照相关操作，已完成的不显示
curl -s -X GET "http://localhost:9200/_snapshot/y_repo/_current"|jq
#通过 stats.processed_files 和 processed_size_in_bytes 估算剩余时间
{
  "snapshots": [
    {
      "snapshot": "snapshot_all",       // 快照名称
      "uuid": "ABC123",                    // 快照唯一ID
      "state": "IN_PROGRESS",              // 执行状态
      "include_global_state": true,        // 是否包含集群全局状态
      "shards_stats": {
        "initializing": 0,                 // 初始化中的分片数
        "started": 5,                      // 进行中的分片数
        "finalizing": 0,                   // 最终化中的分片数
        "done": 10,                        // 已完成的分片数
        "failed": 0,                       // 失败的分片数
        "total": 15                        // 总分片数
      },
      "stats": {
        "number_of_files": 100,            // 总文件数
        "processed_files": 80,             // 已处理文件数
        "total_size_in_bytes": 1024000,    // 总字节数
        "processed_size_in_bytes": 819200  // 已处理字节数
      },
      "indices": {                         // 涉及的索引及各自分片状态
        "logs-2023": {
          "shards_stats": { ... },
          "stats": { ... }
        }
      },
      "start_time": "2023-10-05T14:00:00.000Z",  // 任务开始时间
      "end_time": null                     // 任务结束时间（未完成时为null）
    }
  ]
}

```
# 迁移恢复到其他集群
1. esdump直接导出导入即可不多讲
2. snapshot方式需要将fs的物理存储路径拷贝到新节点，然后在新节点es重新注册同名仓库后快照可以直接来使用恢复；如果是nfs或者s3（自建minio这种）这种类型就不需要迁移文件直接注册同名仓库后恢复即可
