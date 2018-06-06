---
title: "Es"
date: 2018-04-23T19:35:17+08:00
draft: true
tags:
 - java
---
理解和使用es。

### 倒排索引
一个倒排索引由文档中所有不重复词的列表组成，对于其中每一个词，都有一个包含它的文档列表。首先通过分析器，将文档的内容拆分成单独的词，创建一个包含所有不重复词条的列表，然后列出每个词条出现在那个文档。


### doc values 
转置结构在其他系统中经常被称作 _列存储_ 。实质上，它将所有单字段的值存储在单数据列中，这使得对其进行操作是十分高效的，例如排序。

### mapping
在同一个索引中的多个类型中，如果对应的属性名字是一样的，如broker中的name和channel中的name，那么，他们是不能使用不同的类型的，也就是说在同一个索引中，同一个名称只能使用相同的映射类型。


### aggregation


### scroll
在es中，搜索一般包括两个阶段，query和fetch。query确定要取那些doc，fetch则取出具体的doc。假设集群有三个节点。query阶段，client发送请求到node1，node1创建一个from+size的优先级队列用于保存结果，并称为coordinating node，然后广播请求到所有涉及到的shards。每个shard在内部执行搜索请求，并将结果保存到大小同样为from+size的优先级队列中。最后，所有的shard把暂存在自身的数据返回给coordinating node，coordinating node将所有返回的结果进行一次全局合并，然后存入自己的优先级队列。假设有6个shard，则coordinating node将拿到(from+size)*6条数据，合并并排序后，选择前from+size条数据放入自己的优先队列中。

fetch阶段。coordinating node发送get请求到相关shards，shards根据_id获取数据详情返回给coordinating node，最后再返回给client。

根据上面的描述，深度分页会有严重的性能问题。es提供了scroll查询来避免该问题。每次从固定的偏移量获取size个doc，从而coordinating node在每次查询中汇总和排序的doc数量都为shards*size。类似于从传统的order by t.id limit :from,:size转变到了where t.id > :lastMaxId order by t.id limit :size。


### 使用curl操作es
```java
//mapping
curl 10.100.2.50:9200/customer/_mapping/base?pretty --socks5 192.168.40.230:1080

//search
curl 127.0.0.1:9200/customer/base/_search?pretty -d'{"query":{"ids":{"values":["100000858"]}}}'
curl 10.100.2.50:9200/customer/broker/_search?pretty --socks5 192.168.40.230:1080 -d'{"from":0,"size":10,"query":{"bool":{"must_not":[{"range":{"updateTime":{"from":0}}}]}}}'
curl -XGET --socks5 192.168.40.230:1080 10.100.2.50:9200/customer/broker/_search?pretty -d'{"query":{"bool":{"must_not":{"exists":{"field":"updateTime"}}}},"size":10,"from":0}'
curl  --socks5 192.168.40.230:1080 10.100.2.50:9200/customer/base/_validate/query?explain -d'{"query":{"nickName":{"match":"zhangsan"}}}'
curl 10.100.2.50:9200/customer/broker/_search?pretty --socks5 192.168.40.230:1080 -d'{"from":0,"size":10,"query":{"has_parent":{"type":"base","query":{"ids":{"values":[100000679]}}}}}'

//analyze
curl 10.100.2.50:9200/_analyze?pretty --socks5 192.168.40.230:1080 -d'{"analyzer":"standard","text":"Text to analyze"}'

//setting
curl -XPUT http://10.100.2.50:9200/customer/_settings --socks5 192.168.40.230:1080 -d'{"index":{"max_result_window":"2147483647"}}'
curl 10.100.2.50:9200/customer/_settings?pretty --socks5 192.168.40.230:1080

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

es默认将尽可能多的数据缓存到内存中，并且大块的数据是直接保存到老年代的，如果服务器内存不足，将会大致大量的gc，从而导致es服务器响应超时，可以调控es缓存数据的比例，indices.fielddata.cache.size=40%，来防止缓存数据过多导致的gc，但这个比例需要合适，不然会导致大量的io。查看es服务器jvm的内存情况，jstat -gcutil pid 1000。


### ref
- [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
