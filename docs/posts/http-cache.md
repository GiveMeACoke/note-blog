---
lang: zh-CN
title: Http缓存机制
description: Http缓存机制
date: 2024-05-30
Category: ["网络"]
tag: ["http"]
---

![缓存机制](/images/浏览器缓存机制.png)

## 是什么？

Http 缓存机制适用于规定浏览器对于请求资源是否读取缓存的规则
总共分为**不缓存**、**协商缓存**、与**强制缓存**三种缓存机制.

## 具体实现

### 不缓存

设置`Cache-Control`响应头为**no-store**、
浏览器不再读取缓存，每一次请求都会访问服务器

### 协商缓存

#### ETag 头部字段

`ETag` 又叫实体标签，是由服务器生成并非配给每一个资源的唯一标识。

1. 浏览器首次请求资源的时候服务器会在响应头中包含一个`ETag`。浏览器会缓存该资源和`ETag`
2. 当浏览器再一次访问该资源的时候会发送一个带有`If-None-Match`头的条件请求，其中包含先前收到的`ETag`.
3. 服务器处理请求，检查`If-None-Match`和当前的`ETag`是否匹配
   - 匹配，返回`304 Not Modified` 响应，表面资源未修改，浏览器可以使用缓存的副本。
   - 不匹配，返回新的资源和新的`ETag`，浏览器更新缓存

#### Last-Modified 头部字段

`Last-Modified` 头部字段指的是资源的最后修改时间。

1. 客户端在首次请求资源的时候，服务端会在响应头中包含`Last-Modified`字段，浏览器会缓存该资源和`Last-Modified`的时间。
2. 当浏览器再次访问该资源的时候会发送一个带有`If-Modified-Since`的条件请求，其中包含了之前缓存的`Last-Modified`的时间。
3. 服务器通过比较当前`Last-Modified`的时间是否在`If-Modifed-Since`之后：
   - 否，说明资源未修改，返回`304 Not Modified` 浏览器读取缓存资源
   - 是，说明资源已经修改，返回新的资源和新的`Last-Modified`时间

### 强制缓存

#### Expires 头部字段

`Expires` 头部指定资源过期的具体时间点，浏览器在该时间点之前会使用缓存，而不请求服务器。

#### Cache-Control: max-age=

`Cache-Controll:max-age=2592000`，通过`Cache-Control`头部字段设置`max-age`实现强制缓存，它的单位是秒，指定资源从请求时间开始的缓存时间。

## 实际运用场景

对于 Web 项目资源来说，我们可以对应用入口的 html 文件进行协商缓存，对其他在 html 文件中引用到的资源文件进行强制缓存，只要 html 文件不变，他引用的资源就能匹配强制缓存规则，提高页面加载速度.

nginx 的具体实现：

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    server {
        listen       80;
        server_name  example.com;

        # 启用压缩
        gzip on;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        # 对 HTML 文件进行协商缓存
        location ~* \.html$ {
            root /path/to/your/content;
            expires -1;  # 禁用强制缓存
            add_header Cache-Control "no-cache, must-revalidate";  # 协商缓存

            # Optional: Add ETag and Last-Modified headers
            etag on;
            if_modified_since exact;
        }

        # 对其他静态资源文件进行强制缓存
        location ~* \.(css|js|jpg|jpeg|png|gif|svg|ico|woff|woff2|ttf|eot)$ {
            root /path/to/your/content;
            expires 30d;  # 强制缓存 30 天
            add_header Cache-Control "public, max-age=2592000";  # 30 天（2592000 秒）
        }

        error_page 404 /404.html;
        location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
}

```
