---
title: Django学习笔记（二）
date: 2023-09-15 23:31:52
tags: Django
---
# Django缓存

## 基本介绍

动态网站的基本权衡是，嗯，它们是动态的。每 当用户请求页面时，Web 服务器会进行各种计算—— 从数据库查询到模板呈现再到业务逻辑 -- 创建 您网站的访问者看到的页面。这要贵得多，从一个 处理开销透视，比您的标准
从文件系统服务器中读取文件。

对于大多数 Web 应用程序来说，这种开销没什么大不了的。大多数网络 应用程序不是 OR ;他们很小—— 到流量一般的中型网站。但对于中高流量 站点，必须尽可能多地削减开销。 这就是缓存的用武之地。

缓存某些东西就是保存昂贵计算的结果，以便 下次不必执行计算。这是一些伪代码 解释这将如何用于动态生成的网页：

```pycon
given a URL, try finding that page in the cache
if the page is in the cache:
    return the cached page
else:
    generate the page
    save the generated page in the cache (for next time)
    return the generated page
```

## 基本使用

### 设置缓存的介质

django框架提供了多种缓存介质用于缓存的那些经常不变化的数据，常见的有以下几种：

1. 数据库缓存

尽管的存储介质没有更换，但是会把一次负责查询的结果直接存储到表中，例如多个条件过滤的结果，可以避免重复查询，提升效率。
配置如下：

```pycon
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.db.DatabaseCache",
        "LOCATION": "my_cache_table", # 缓存的表名
        "TIMEOUT":300, # 设置缓存的时间，单位秒
        "POTIONS":{
            "MAX_ENTRIES":300, # 缓存最大数据条数
            "CULL_FREQUENCY":2 # 缓存达到最大数据条时，删除的数据量 1/x，此处为 1/2
        }
    }
}
```

执行命令 python manage.py createcachetable 这会在数据库中创建一个表，名字由`LOCATION`决定。

2. 本地内存缓存（测试使用）

配置如下：

```pycon
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
        "LOCATION": "unique-snowflake",
    }
}
```

3. 本地文件缓存

配置如下：

```pycon
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.filebased.FileBasedCache",
        "LOCATION": "/var/tmp/django_cache", # 文件夹路径
        #  "LOCATION": "c:/foo/bar", windows下
    }
}
```

4. redis内存型数据库缓存（后面详细介绍）

配置如下：

```pycon
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379",
    }
}
```

### 缓存表结构

使用数据库缓存一般创建出来的表结构为：
field&emsp;&emsp;&emsp;&emsp;&emsp;type  
cache_key&emsp;&emsp;varchar(255)  
value&emsp;&emsp;&emsp;&emsp;&nbsp; longtext  
expires&emsp;&emsp;&emsp;&nbsp; datetime(6)

### 整体缓存策略

#### 视图函数

使用Django提供的`cache_page`装饰器,可以装饰某个视图函数,并且将这个函数加入到缓存中

```pycon
from django.http import HttpResponse
import time
from django.views.decorators.cache import cache_page
# 传入的参数为过期时间,单位为S
@cache_page(15)
def cache_test(request):
    t = time.time()
    return HttpResponse(f'{t}')
```

#### 路由

如果不在意视图函数的细节,我们可以直接在路由层面来进行缓存,如下:

```pycon
from django.urls import path
from . import views
from django.views.decorators.cache import cache_page

app_name = 'front'
urlpatterns = [
    path('', cache_page(15)(views.cache_test), name='test')
]

```

### 局部缓存策略

相比于上面的视图缓存,粒度更加细化。

方式一：
使用`caches`加载一个缓存配置项，得到一个`cache`缓存对象，此时可以通过缓存对象存储数据。如下：

```pycon
# settings 文件
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.db.DatabaseCache",
        "LOCATION": "my_cache_table",
        "TIMEOUT": 300,  # 设置缓存的时间，单位秒
        "POTIONS": {
            "MAX_ENTRIES": 300,  # 缓存最大数据条数
            "CULL_FREQUENCY": 2  # 缓存达到最大数据条时，删除的数据量 1/x，此处为 1/2
        }
    },
    "front": {
        "BACKEND": "django.core.cache.backends.db.DatabaseCache",
        "LOCATION": "my_cache_table",
        "TIMEOUT": 300,  # 设置缓存的时间，单位秒
        "POTIONS": {
            "MAX_ENTRIES": 300,  # 缓存最大数据条数
            "CULL_FREQUENCY": 2  # 缓存达到最大数据条时，删除的数据量 1/x，此处为 1/2
        }
    }
}

# views.py
from django.core.cache import caches
def cache_test(request):
    cache = caches['front']
```

方式二：
使用`cache`无需传递配置名，直接加载default选项配置的缓存

```pycon
from django.core.cache import cache

def cache_test(request):
    pass
```

#### cache的方法

1. set  
   第一个参数为key,第二个为具体的value,第三个是过期时间

```pycon
cache.set('my_key', 'hello, world!', 30)
```

2. get  
   传递一个key值得到对应的缓存数据，如果缓存的数据消失，那么此时返回None

```pycon
cache.get("my_key")
```

3. add  
   添加一个键值对数据的到缓存，与set不同，add只有当key不存在的时候才能成功设置值

```pycon
cache.add("add_key", "New value")
```

4. get_or_set  
   如果获取的值不存在则执行set
5. set_many  
   批量存储键值对数据
6. set_many  
   批量设置数据
7. delete  
   删除成功返回True否则返回False

```pycon
cache.delete("a")
```

8. delete_many  
   批量删除数据

### 浏览器缓存

#### 强缓存

使用响应头`Cache-Control`，浏览器会缓存当前网页，下次请求的时候就不会请求服务器，而是直接从本地缓存中读取网页。

```http-header
Cache-Control:max-age:120
```

上面代表浏览器缓存120秒，失效后才会去重新向服务器拿取资源。  
在Django中如果我们使用了`cache_page`装饰器，那么在响应头中会自动添加Cache-Control，很方便。

#### 协商缓存

主要针对图片，视频等大的文件，图片等资源不易变化，到期后浏览器会跟服务器协商，查看当前缓存是否可用，如果可用那么此时服务器不必返回数据，否则返回最新图片数据即可。  
协商缓存在强缓存的基础上来完成，靠`Last-Modified`头完成（值为最近修改时间），服务器返回此头信息则代表此文件需要协商缓存。图片到期，浏览器将`Last-Modified`的值作为
`If-Last-Modified`请求头的值，发送给服务器协商是否需要更新，如果服务器返回`304`则代表继续缓存，返回`200`代表需要更新，并且响应体为最新的资源。

现在主流协商方案使用的是`ETag`请求头和`If-None-Match`请求头，此方案靠Hash值来判断文件是否产生了变化。此响应头优先级大于`Last-Modified`。
