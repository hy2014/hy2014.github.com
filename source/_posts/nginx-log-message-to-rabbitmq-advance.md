title: log nginx request to rabbitmq 进阶篇
date: 2014-10-14 17:29:32
tags: nginx
description: "Nginx发送Message到rabbitmq进阶篇"
toc: true
category: nginx
comment: true
---

本文尝试提供另外一种思路, 提高nginx发送message到rabbitmq的性能.

<!-- more -->

###前言
上篇[博客][1]的代码逻辑是很简单的

> 1. nginx接受请求
> 2. 组装数据
> 3. 建立RabbitMQ的连接, 将组装好的数据发送到RabbitMQ中。

可以看出, 这种方式是很低效的, 每一个请求都创建一个新的连接, 就算我们设置share connection pool, 但是在我的dev环境下,使用ab做性能测试.

	ab -n 10000 -c 300 -p postdata ${url}

当并发超过300的时候, 连接rabbitmq就会频繁失败, 因此我们需要另外一种思路.

##Openresty - Timer & Share Dict
首先简单认识openresty的lua module中, 提供的两个功能函数

> Timer - 提供了类似调度器的功能
> Share Dict - 一种可以在Worker进程共享的Cache.

关于Timer和Share Dict的更多介绍, 请读者访问官方[文档][2].

##基本的思路
> 1. 将接受的请求组装成数据
> 2. 将数据保存到Share Dict中
> 3. 后台运行timer, 定期检查, 在特定条件下, 批量发送Share Dict的数据到rabbitmq中.

##实现过程
我们需要将Share Dict做一个简单的封装, 变成一个Queue的模型, 供Timer去调用. 在这里, 我们将timer将消息批量发送到RabbitMq的过程称之为flush阶段.
先介绍两个variable
	
	write_point - 表示累计接收的request count。
	read_point - 表示timer累计从cache中读取的request count。

整个流程分为两个阶段

> 接收请求阶段: 接收一个请求时，将write_point原子性自增1，将返回的, 已经更新过的write_point值作为存放当前request data的key, 存放到Share Dict中.
> Timer运行阶段: Timer启动后, 读取write_point和read_point, 如果发现read_point<write_point, 开启flush阶段, 不断自增read_point, 将新的read_point值作为key, 从Share Dict取出data, 发送到rabbitmq中, 直到read_point=write_point, 此次flush工作结束. 

所以我们通过这种方式, 从一开始的每条消息创建一个Connection, 变成了一次flush批量的消息, 消耗一个Connection, 这样也解决了前面遇到的性能瓶颈.

##注意事项和建议
在这里列出一些我认为可以进一步增强该模块稳定性的建议

> 使用Openresty的Lock, 保证当前只有一个timer在进行flush工作.
> 对Timer的调度逻辑进行适量优化, 例如可以增加一个变量存放当前cache中request的容量, 当一次flush结束后, 尝试检查该容量值, 一旦容量超出radio, timer可以立即启动, 否则定时启动.
> 增加一个monitor的url， 管理人员可以通过url获取关键Metric, 获取当前系统的健康状况.

  [1]: http://hy2014.github.io/2014/09/24/log-nginx-request-to-rabbitmq/
  [2]: http://wiki.nginx.org/HttpLuaModule