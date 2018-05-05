---
title: "Es"
date: 2018-04-23T19:35:17+08:00
draft: true
---

理解和使用es。


```java
#数据查询
curl 127.0.0.1:9200/customer/base/_search?pretty -d'{"query":{"ids":{"values":["100000858"]}}}'
#设置
curl -XPUT http://10.100.2.50:9200/customer/_settings --socks5 192.168.40.230:1080 -d'{"index":{"max_result_window":"2147483647"}}'
#查看集群状态
curl -XGET localhost:9200/_cat 
curl -XGET localhost:9200/_cat/indices?v 
curl -XGET localhost:9200/_cluster/stats
curl -XGET localhost:9200/_nodes/stats
```

由于大量垃圾数据的存在，在更新子文档的时候，没有对存在性进行判断，导致es服务器打印大量文档丢失的日志，并引发monitor对数据进行收集，从而导致es服务器由于内存不足而挂掉。

### ref
- [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
