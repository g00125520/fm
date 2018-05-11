---
title: "Es"
date: 2018-04-23T19:35:17+08:00
draft: true
---
理解和使用es。


### 倒排索引


### mapping


### aggregation

### scroll
在es中，搜索一般包括两个阶段，query和fetch。query确定要取那些doc，fetch则取出具体的doc。假设集群有三个节点。query阶段，client发送请求到node1，node1创建一个from+size的优先级队列用于保存结果，并称为coordinating node，然后广播请求到所有涉及到的shards。每个shard在内部执行搜索请求，并将结果保存到大小同样为from+size的优先级队列中。最后，所有的shard把暂存在自身的数据返回给coordinating node，coordinating node将所有返回的结果进行一次全局合并，然后存入自己的优先级队列。假设有6个shard，则coordinating node将拿到(from+size)*6条数据，合并并排序后，选择前from+size条数据放入自己的优先队列中。

fetch阶段。coordinating node发送get请求到相关shards，shards根据_id获取数据详情返回给coordinating node，最后再返回给client。

根据上面的描述，深度分页会有严重的性能问题。es提供了scroll查询来避免该问题。每次从固定的偏移量获取size个doc，从而coordinating node在每次查询中汇总和排序的doc数量都为shards*size。类似于从传统的order by t.id limit :from,:size转变到了where t.id > :lastMaxId order by t.id limit :size。


### 使用curl操作es
```java
//search
curl 127.0.0.1:9200/customer/base/_search?pretty -d'{"query":{"ids":{"values":["100000858"]}}}'
//setting
curl -XPUT http://10.100.2.50:9200/customer/_settings --socks5 192.168.40.230:1080 -d'{"index":{"max_result_window":"2147483647"}}'
//status
curl -XGET localhost:9200/_cat 
curl -XGET localhost:9200/_cat/indices?v 
curl -XGET localhost:9200/_cluster/stats
curl -XGET localhost:9200/_nodes/stats
//scroll
curl 127.0.0.1:9200/customer/base/_search?pretty&scroll=1m -d'{"query":{"match":{"nickName":"zhangsan"}}}'
curl 127.0.0.1:9200/_search/scroll -d'{"scroll":"1m","scroll_id":"AAAXX"}'
```

### 坑

由于大量垃圾数据的存在，在更新子文档的时候，没有对存在性进行判断，导致es服务器打印大量文档丢失的日志，并引发monitor对数据进行收集，从而导致es服务器由于内存不足而挂掉。

### ref
- [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
