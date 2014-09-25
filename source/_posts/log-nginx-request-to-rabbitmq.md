title: log nginx request to rabbitmq
date: 2014-09-24 14:27:38
tags: nginx
description: "Use Lua & Nginx send log to rabbitmq"
toc: true
category: nginx
---

花了点时间research, 如何使用nginx和Lua将数据请求发送到Rabbitmq中。

本文涉及的知识点:

> 1. access_log如何log POST request body(系统必须支持POST的请求)。

> 2. 通过使用openresty, 编写Lua脚本将数据发送到Rabbitmq中。

<!-- more -->

###OpenResty
OpenResty（也称为 ngx_openresty）是一个全功能的 Web 应用服务器，它打包了标准的 Nginx 核心，很多的常用的第三方模块，以及它们的大多数依赖项， 具体内容另行Google。

###Lua
Lua脚本是一个很轻量级的脚本，也是号称性能最高的脚本。Openresty有一个nginx_lua模块，它能将lua语言嵌入到nginx配置中,从而使用lua就极大增强了nginx的能力(想像一下用C写nginx模块， 现在只需要使用Lua来实现, 生产效率会得到极大提高)。

###Enable Nginx log POST Request
Nginx本来默认是不支持将post请求的request body 记录到日志中，但是这个[链接][1], 有很多方法实现这个需求, 这里介绍两种方案。

> 通过下面的配置使Lua脚本中能够访问$request_body。

```
location 1.gif {
    lua_need_request_body on;
}
```

> 在Nginx的配置, 使$request_body变量有意义

```
location 1.gif {
    echo_read_request_body;
}
```
##Lua visit Rabbitmq
通过该[链接][2]， 可以看到已经有个实现了Lua访问Rabbitmq的client library, 可以将rabbitmqstomp.lua拷贝到${OpenResty_HOME}/lualib下。

最后nginx.conf内容大致如下
```
log_format post_format  '$remote_addr - $request_body';
location 1.gif {
    default_type image/gif;
    lua_need_request_body on;
    #这里关掉access_log功能
    access_log off; 

    access_by_lua_file xx.lua;
    empty_gif;
}
location /i-log { 
    internal;
    log_subrequest on; 
    access_log logs/xx.log post_format;
    echo_read_request_body;
    echo ''; 
}
```

> 这里需要提醒的是, 如果request_body中带有换行等特殊字符，Nginx在写入日志文件之前进行escape.


Lua脚本的内容如下:
```lua
	local rabbitmq = require "rabbitmqstomp"
    local mq, err = rabbitmq:new()
		   
    if not mq then
      ...
    end
    
    mq:set_timeout(60000)
    
    local ok, err = mq:connect {
        host = "192.168.225.209",
        port = 61613,
        username = "guest",
        password = "guest",
        vhost = "/"
    }           
    if not ok then
      ...
    end
    
    local post_body = ngx.unescape_uri(ngx.var.request_body)
    
    # the message content sent to queue.
    local message = ..... 
    local headers = {}
    headers["destination"] = "/queue/extractor_test_queue"
    headers["receipt"] = "msg#1"
    headers["content-type"] = "text/plain"

    local ok, err = mq:send(data, headers)
    if not ok then
        ...
    end
    #使用subrequest to 记录access_log
    ngx.location.capture("/i-log", {body = ngx.var.request_body})
```

到此为止， 基本上实现了使用Lua和OpenResty将数据发送到Rabbitmq中， 下一步关注的是该实现方式的性能和可靠性。

  [1]: http://stackoverflow.com/questions/4939382/logging-post-data-from-request-body
  [2]: https://github.com/wingify/lua-resty-rabbitmqstomp
