---
layout:  post
title:		缓存架构之nginx+lua+java完善三级缓存
subtitle:		缓存架构之nginx+lua+java完善三级缓存 
date:     2018-07-04
author:   Fogeeks
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Redis
    - Nginx
    - Lua
---
 
#	缓存架构之nginx+lua+java完善三级缓存 

多级缓存最后一步nginx缓存，当有读请求过来时候，根据如商品id和商品店铺id，在nginx本地缓存中尝试获取数据，
如果在nginx本地缓存中没有获取到数据，那么就到redis分布式缓存中获取数据，
如果获取到了数据，还要设置到nginx本地缓存中，

但是这里有个问题，建议不要用nginx+lua直接去获取redis数据，因为**openresty**没有太好的redis cluster的支持包，所以建议是发送http请求到缓存数据生产服务，由该服务提供一个http接口，缓存数生产服务可以基于redis cluster api从redis中直接获取数据，并返回给nginx，

如果缓存数据生产服务没有在redis分布式缓存中没有获取到数据，那么就在自己本地ehcache中获取数据，返回数据给nginx，也要设置到nginx本地缓存中， 
如果ehcache本地缓存都没有数据，那么就需要去原始的服务中拉去数据，该服务会从mysql中查询，拉去到数据之后，返回给nginx，并重新设置到ehcache和redis中。

nginx最终利用获取到的数据，**动态渲染网页模板**


这里的lua脚本可以这样写

```javascript

local uri_args = ngx.req.get_uri_args()
local productId = uri_args["productId"]
local shopId = uri_args["shopId"]

local cache_ngx = ngx.shared.my_cache

local productCacheKey = "product_info_"..productId
local shopCacheKey = "shop_info_"..shopId

local productCache = cache_ngx:get(productCacheKey)
local shopCache = cache_ngx:get(shopCacheKey)

if productCache == "" or productCache == nil then
	local http = require("resty.http")
	local httpc = http.new()
	local resp, err = httpc:request_uri("http://192.168.31.179:8080",{
  		method = "GET",
  		path = "/getProductInfo?productId="..productId
	})
	productCache = resp.body
	cache_ngx:set(productCacheKey, productCache, 10 * 60)
end

if shopCache == "" or shopCache == nil then
	local http = require("resty.http")
	local httpc = http.new()
	local resp, err = httpc:request_uri("http://192.168.31.179:8080",{
  		method = "GET",
  		path = "/getShopInfo?shopId="..shopId
	})
	shopCache = resp.body
	cache_ngx:set(shopCacheKey, shopCache, 10 * 60)
end

local cjson = require("cjson")
local productCacheJSON = cjson.decode(productCache)
local shopCacheJSON = cjson.decode(shopCache)

local context = {
	productId = productCacheJSON.id,
	productName = productCacheJSON.name,
	productPrice = productCacheJSON.price,
	productPictureList = productCacheJSON.pictureList,
	productSpecification = productCacheJSON.specification,
	productService = productCacheJSON.service,
	productColor = productCacheJSON.color,
	productSize = productCacheJSON.size,
	shopId = shopCacheJSON.id,
	shopName = shopCacheJSON.name,
	shopLevel = shopCacheJSON.level,
	shopGoodCommentRate = shopCacheJSON.goodCommentRate
}

local template = require("resty.template")
template.render("product.html", context)

```


下面可以看下测试结果

`` 想nginx发送数据生产服务变更请求（简化版）``

> wget http://192.168.111.130/lua?productId=111&shopId=211

```html
<html>
        <head>
                <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
                <title>商品详情页</title>
        </head>
<body>
商品id: 111<br/>
商品名称: 模拟MySQL取出来的数据-iphone7手机<br/>
商品图片列表: a.jpg,b.jpg<br/>
商品规格: 测试-iphone7的规格<br/>
商品售后服务: 测试-iphone7的售后服务<br/>
商品颜色: 测试-红色,白色,黑色<br/>
商品大小: 5.5<br/>
店铺id: 211<br/>
店铺名称: 模拟MySQL取出来的数据--苹果旗舰店<br/>
店铺等级: 5<br/>
店铺好评率: 0.99<br/>
</body>
</html>

```
 
 `` nginx没有缓存该数据的话，请求就会打入数据生产服务后台，它接收到请求后依次尝试查询redis,ehcache，mysql,在mysql查到数据之后还会刷新缓存 ``
 
```text
=================从redis中获取缓存，商品信息=null
 从本地ehcache缓存中获取商品信息：111
=================从ehcache中获取缓存，商品信息=null
=================从mysql中获取数据并刷新到缓存，商品信息=ProductInfo [id=111, name=模拟MySQL取出来的数据-iphone7手机, price=5599.0, pictureList=a.jpg,b.jpg, specification=测试-iphone7的规格, service=测试-iphone7的售后服务, color=测试-红色,白色,黑色, size=5.5, shopId=1, modifiedTime=2018-01-01 12:01:00]
=================从redis中获取缓存，店铺信息=null
 从本地ehcache缓存中获取店铺信息：211
=================从ehcache中获取缓存，店铺信息=null
 将店铺信息保存到本地的ehcache缓存中：ShopInfo [id=211, name=模拟MySQL取出来的数据--苹果旗舰店, level=5, goodCommentRate=0.99]
将店铺信息保存到redis中：ShopInfo [id=211, name=模拟MySQL取出来的数据--苹果旗舰店, level=5, goodCommentRate=0.99]
=================从mysql中获取数据并刷新到缓存，店铺信息=ShopInfo [id=211, name=模拟MySQL取出来的数据--苹果旗舰店, level=5, goodCommentRate=0.99]
success to acquire lock for product[id=111]
existed product info is null......
  将商品信息保存到本地的ehcache缓存中：ProductInfo [id=111, name=模拟MySQL取出来的数据-iphone7手机, price=5599.0, pictureList=a.jpg,b.jpg, specification=测试-iphone7的规格, service=测试-iphone7的售后服务, color=测试-红色,白色,黑色, size=5.5, shopId=1, modifiedTime=2018-01-01 12:01:00]
将商品信息保存到redis中：ProductInfo [id=111, name=模拟MySQL取出来的数据-iphone7手机, price=5599.0, pictureList=a.jpg,b.jpg, specification=测试-iphone7的规格, service=测试-iphone7的售后服务, color=测试-红色,白色,黑色, size=5.5, shopId=1, modifiedTime=2018-01-01 12:01:00]
release the lock for product[id=111]......

```

然而把该请求再次发起时，后台没有任何反应，前端就有了数据，这就说明nginx已经缓存了该数据





 