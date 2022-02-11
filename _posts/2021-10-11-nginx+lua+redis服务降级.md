---
layout:     post
title:      nginx+lua+redis服务降级
subtitle:   Lua
date:       2021-10-11
author:     果果
header-img: img/post-bg-mma-3.jpg
catalog: false
tags:
- 学习资料
---

## 1 引子
各位有没有过这样的经历：
```
商城或web站点的用户访问量出乎意料地增加了很多，超出了系统的负载能力， 系统有些扛不住，继而导致注
册，下单，支付什么的全部在绕圈卡住，继而导致公司业务损失了不少用户和订单。
```

## 2 什么是降级, 为什么降级，降级的场景 
降级的最终目的是保证核心服务的**高可用**。过程就是丢卒保帅，有些服务是无法降级的，比如支付。<br/>
当我们的服务器压力剧增为了保证核心功能的可用性 ，而选择性的降低一些功能的可用性，或者直接关闭该功能。<br/>
这就是典型的丢车保帅了。 就比如贴吧类型的网站，当服务器吃不消的时候，可以选择把发帖功能关闭，注册功能<br/>
关闭，改密码，改头像这些都关了，为了确保登录和浏览帖子这种核心的功能。<br/>

**降级的原理**：就是降低次要功能的可用性实用性，增加核心功能的高可用性。<br/>

## 3 降级的种类
    3.1 根据降级的开关位置：分为服务代码降级和开关前置降级。

![fw1](/img-post/202110/fw1.jpg "fw1")

    
```
3.2 根据读写：分为读降级和写降级。
读降级，比如，读取动态数据，降级为读取静态数据。
写降级，比如，写入mysql，降级为写入消息队列， 等高峰期过后，在从队列写入mysql。
```

```
3.3 根据降级的性质：分为返回内容降级，限流降级，限速降级。
返回内容降级，比如，返回实时数据，降级为返回兜底数据
限流降级，比如，1000个请求，我只接受500个。 这么做也是无奈之下选择，要不然系统崩了 谁都访问不了
限速降级，比如，对于那些访问过于频繁的ip进行限速

*Tips   nginx 自带限流限速，比如ngx_http_limit_req_module和ngx_http_limit_conn_module 。
```

```
3.4 根据降级的维护特点：分为手动降级和自动降级。
手动降级，是人为看到系统负载异常后，手动调整降级
自动降级，是系统监测到异常后，自动降级，自动降级虽然更加智能，但有时候自动脚本可能会干一些超乎预
料的事情。
```
## 4 降级线路图 

![fw2](/img-post/202110/fw2.jpg "fw2")

## 5 实现降级
### 5.1 代码流程图：
![fw3](/img-post/202110/fw3.jpg "fw3")

### 5.2 配置中心：
配置中心是一个后台，这个配置中心大家根据各自需求自己去实现

### 5.3 nginx+lua+redis实现降级：

nginx配置文件/etc/nginx/nginx.conf

```nginx
user  nobody;
worker_processes  1;

events {
    worker_connections  1024;
}

http {

    #/usr/share/lua/5.1/lua-resty-limit-traffic-master/lib/?.lua;;
    lua_package_path "/usr/share/lua/5.1/lua-resty-limit-traffic-master/lib/?.lua;;/usr/share/lua/5.1/lua-resty-redis/lib/?.lua;;/usr/share/lua/5.1/lua-resty-redis-cluster/lib/resty‘7/?.lua;;";

    lua_package_cpath "/usr/share/lua/5.1/lua-resty-redis-cluster/lib/libredis_slot.so;;";

    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;
	lua_shared_dict my_limit_req_store 100m;
	
    server {
        listen       80;
        server_name  127.0.0.1;
        
        #获取广告推荐数据
	    location /goods_list_advert {
	        default_type 'application/x-javascript;charset=utf-8';
            content_by_lua_file /etc/nginx/lua/goods_list_advert.lua;
        }

        #从服务层+mysql获取数据
        location /goods_list_advert_from_data {
            #allow   127.0.0.1;
            #deny    all;

            default_type 'application/x-javascript;charset=utf-8';
            #rewrite https//www.baidu.com/ break;
            proxy_pass   http://192.168.232.101:8080/goods/getList;
            #content_by_lua '
             #   ngx.say("从服务层+mysql获取数据")
            #';
        }
    }
}

```

lua降级代码：/etc/nginx/lua/goods_list_advert.lua
```lua
--获取get或post参数--------------------

local request_method = ngx.var.request_method
local args = nil
local param = nil
--获取参数的值
if "GET" == request_method then
    args = ngx.req.get_uri_args()
elseif "POST" == request_method then
    ngx.req.read_body()
    args = ngx.req.get_post_args()
end
sku_id = args["sku_id"]


--关闭redis的函数--------------------

local function close_redis(redis_instance)
    if not redis_instance then
        return
    end
    local ok,err = redis_instance:close();
    if not ok then
        ngx.say("close redis error : ",err);
    end
end


--连接redis--------------------

local redis = require("resty.redis");
--local redis = require "redis"
-- 创建一个redis对象实例。在失败，返回nil和描述错误的字符串的情况下
local redis_instance = redis:new();
--设置后续操作的超时（以毫秒为单位）保护，包括connect方法
redis_instance:set_timeout(1000)
--建立连接
local ip = '127.0.0.1'
local port = 6379
--尝试连接到redis服务器正在侦听的远程主机和端口
local ok,err = redis_instance:connect(ip,port)
if not ok then
    ngx.say("connect redis error : ",err)
    return close_redis(redis_instance);
end

--从redis里面读取开关--------------------

local key = "level_goods_list_advert"
local switch, err = redis_instance:get(key)
if not switch then
    ngx.say("get msg error : ", err)
    return close_redis(redis_instance)
end


--得到的开关为空处理--------------------

if switch == ngx.null then
    switch = "FROM_DATA"  --比如默认值
end


--当开关是要从服务中获取数据时--------------------
if "FROM_DATA" == switch then
    ngx.exec('/goods_list_advert_from_data');

--当开关是要从缓存中获取数据时--------------------
elseif "FROM_CACHE" == switch then

    local resp, err = redis_instance:get("redis_goods_list_advert")
    ngx.say(resp)

--当开关是要从静态资源中获取数据时--------------------
elseif "FROM_STATIC" == switch then

    ngx.header.content_type="application/x-javascript;charset=utf-8"
    local file = "/etc/nginx/html/goods_list_advert.json"
    local f = io.open(file, "rb")
    local content = f:read("*all")
    f:close()
    ngx.print(content)

--当开关是要停掉数据获取时--------------------
elseif "SHUT_DOWN" == switch then

    ngx.say('no data')
end

--close_redis(redis_instance)
--ngx.exit(ngx.OK)

close_redis(redis_instance)
```

## 6 自动降级
原理：采用nginx+lua+redis对错误返回进行统计，当统计到的错误次数达到一定数值时，lua会判断这个值， 然后
处理

```lua
--判断错误的响应，并进行计数， 后续便可以参考这个数值进行降级
if tonumber(ngx.var.status) ~= 200 then
    ngx.say(ngx.var.status)
    ngx.log(ngx.ERR,"upstream reponse status is " .. ngx.var.status .. ",please notice it")

    local error_count, err = redis_instance:get("error_count_goods_list_advert")
    --得到的数据为空处理
    if error_count == ngx.null then
        error_count = 0
    end
    error_count = error_count + 1

    --统计错误次数到error_count_goods_list_advert
    local resp,err = redis_instance:set("error_count_goods_list_advert",error_count)
    if not resp then
        ngx.say("set msg error : ",err)
        return close_redis(redis_instance)
    end
end
```

## 7 自动降级（漏桶原理）
### 7.1 实现目标：
nginx+lua实现的多级缓存降级，可以在多种降级方案（mysql,nosql,静态文件）中自由切换，配备多个出水
口，这样桶的容量和每秒出水量都增加了不少。

![fw4](/img-post/202110/fw4.jpg "fw4")
### 7.2 实现
```
-- git下载自动降级的 lua-resty-limit-traffic 模块

https://github.com/openresty/lua-resty-limit-traffic
```

```lua
--获取get或post参数--------------------

local request_method = ngx.var.request_method
local args = nil
local param = nil
--获取参数的值
if "GET" == request_method then
    args = ngx.req.get_uri_args()
elseif "POST" == request_method then
    ngx.req.read_body()
    args = ngx.req.get_post_args()
end
sku_id = args["sku_id"]


--关闭redis的函数--------------------

local function close_redis(redis_instance)
    if not redis_instance then
        return
    end
    local ok,err = redis_instance:close();
    if not ok then
        ngx.say("close redis error : ",err);
    end
end


--连接redis--------------------

local redis = require("resty.redis");
--local redis = require "redis"
-- 创建一个redis对象实例。在失败，返回nil和描述错误的字符串的情况下
local redis_instance = redis:new();
--设置后续操作的超时（以毫秒为单位）保护，包括connect方法
redis_instance:set_timeout(1000)
--建立连接
local ip = '127.0.0.1'
local port = 6379
--尝试连接到redis服务器正在侦听的远程主机和端口
local ok,err = redis_instance:connect(ip,port)
if not ok then
    ngx.say("connect redis error : ",err)
    return close_redis(redis_instance);
end



-- 加载nginx—lua限流模块
local limit_req = require "resty.limit.req"
-- 这里设置rate=50个请求/每秒，漏桶桶容量设置为1000个请求
-- 因为模块中控制粒度为毫秒级别，所以可以做到毫秒级别的平滑处理
local lim, err = limit_req.new("my_limit_req_store", 50, 1000)
if not lim then
    ngx.log(ngx.ERR, "failed to instantiate a resty.limit.req object: ", err)
    return ngx.exit(501)
end

local key = ngx.var.binary_remote_addr
local delay, err = lim:incoming(key, true)


ngx.say("计算出来的延迟时间是：")
ngx.say(delay)

--if ( delay <0 or delay==nil ) then
    --return ngx.exit(502)
--end


-- 1000以外的就溢出
if not delay then
    if err == "rejected" then
        return ngx.say("1000以外的就溢出")
        -- return ngx.exit(502)
    end
    ngx.log(ngx.ERR, "failed to limit req: ", err)
    return ngx.exit(502)
end

-- 50-100的等待从微服务+mysql获取实时数据;（100-50）/50 =1
if ( delay >0 and delay <=1 ) then
    ngx.say("第50-第100个 等待0-1秒后 从mysql获取数据")
    ngx.sleep(delay)

-- 100-400的直接从redis获取实时性略差的数据;（400-50）/50 =7
elseif ( delay >1 and delay <=7 ) then
    local resp, err = redis_instance:get("redis_goods_list_advert")
    ngx.say("第100-第400个 降级为从redis获取数据")
    ngx.say(resp)
    return

-- 400-1000的从静态文件获取实时性非常低的数据（1000-50）/50 =19
elseif ( delay >7) then

    ngx.say("第400-第1000个 降级为从静态文件（死页面）获取数据")
    ngx.header.content_type="application/x-javascript;charset=utf-8"
    local file = "/etc/nginx/html/goods_list_advert.json"
    local f = io.open(file, "rb")
    local content = f:read("*all")
    f:close()
    ngx.print(content)
    return
end


ngx.say("进入查询微服务+mysql（最实时的数据，进入这里就是没降级）")
```
