---
title: "Srouter"
date: 2018-05-17T17:10:56+08:00
draft: true
tags:
 - linux
---

### 机房内跨架构域服务调用

服务a调用服务b，a和b在同一个机房dc1的不同架构域，a把请求发给本机房的srouter，srouter的fabio模块从consul获取服务b的运行信息，并选择发送给其中一个实例。


### 跨机房，跨架构域的调用

上面的b服务迁移到dc2机房后，可以不用修改代码，直接通过srouer进行调用。

1. 通过srouter的fabio模块，注入服务器路由信息给srouter，route add svc-b /foo http://srouter.local.com/ -opts "host=dc2"
2. 服务a发送调用请求到本地srouter；
3. srouter的nginx模块收到请求后，转发给fabio模块；
4. fabio根据自定义路由，又一次将请求转发给本地的srouter，并在header中加上host: dc2；
5. srouter根据配置好的dc2的upstream，将请求转发给dc2的srouter；
6. dc2的srouter转发给本机的fabio；
7. fabio从consul中获取迁移后的服务b的信息，并将请求转发给服务b；


upstream使用nginx的主备upstream实现了流量的自动切换，当专线故障时候ip1/2/3都访问不同，sRouter-nginx自动将到dc2的流量切换到vip上，如果专线恢复，则又切回ip1/2/3。
nginx.conf片段属于每新增一个dc时，要新生成的配置信息，其中ip1/2/3/vip可以通过consul接口获取，所以可以使用工具生成，从而实现自动化。

### 在feign中使用。

直接调用在consul上注册的ebus，本地的ebus服务：@FeignClient(value = "ebus", decode404 = true, configuration = EbusFeignClientConfiguration.class)

通过调用在consul上注册的srouter服务，通过srouter的代理实现跨机房的调用：@FeignClient(value = "sRouter", decode404 = true, configuration = EbusFeignClientConfiguration.class)

### ref
- [nginx request processing](http://nginx.org/en/docs/http/request_processing.html)
