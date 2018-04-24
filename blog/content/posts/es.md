---
title: "Es"
date: 2018-04-23T19:35:17+08:00
draft: true
---

理解和使用es。


```java
#数据查询
curl 127.0.0.1:9200/customer/base/_search?pretty -d'{"query":{"ids":{"values":["100000858"]}}}'
#查看集群状态
curl -XGET localhost:9200/_cat 
curl -XGET localhost:9200/_cluster/stats
curl -XGET localhost:9200/_nodes/stats
```

### ref
- [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
