---
title: Django Cache 的使用
date: 2017-04-23 21:38:38
tags: 
- django
- cache
---



本文主要来源于[django官方文档](https://docs.djangoproject.com/en/1.11/topics/cache/)

原理是：given a URL, try finding that page in the cache
```c
if the page is in the cache:
    return the cached page
else:
    generate the page
    save the generated page in the cache (for next time)
    return the generated page
```
先需要几步setup，决定cache存在哪，比如存在数据库，或文件系统，或内存。但是因为配置内容太多，觉得放在文章后面额部分。[点此调到后方cache配置](#memcached)

## 所以这里先说一下我们实际写代码中最常用的用法 —— cache一个page

```python
from django.views.decorators.cache import cache_page, never_cache

@cache_page(60 * 15)
def my_view(request):

```

cache_page 中传入的参数是timeout，单位是秒，源码当中核心就是

```python
def cache_page(*args, **kwargs):
    return decorator_from_middleware_with_args(CacheMiddleware)(
        cache_timeout=cache_timeout, cache_alias=cache_alias, key_prefix=key_prefix
    )
```

实际发现如果设置了max_age，max_age的级别更高会覆盖timeout，所以普通使用的时候只设置timeout就好（我们猜想这么设计是为了给浏览器的数据和后台server cache失效时间要一致，因为max-age应该是给浏览器用的）

(此处源码在 django/middleware/cache.py:)
```python
class UpdateCacheMiddleware(object):
    def process_response(self, request, response):
        '''此处省略很多代码'''
        timeout = get_max_age(response)
        if timeout is None:
            timeout = self.cache_timeout
```

never_cache也是很好用的一个decorator，相当于把各种参数都设置成不缓存，就是以下效果 

'cache-control': 'no-cache'

<br>
## 还可以较底层的使用方法


```python
from django.core.cache import caches
cache1 = caches['myalias']
```
这样就得到了一个cache的instance。如果不想指明，可以用默认的cache instance：
```python
from django.core.cache import cache
```
这个cache instance相当于是caches['default']。
cache的数据结构就是键值对儿，所以用法是 set(key, value, timeout) 和 get(key)。
```python
cache.set('my_key', 'my_value', 30)
cache.get('my_key')
```
cache还有很多方法，比如add，get_or_set，delete等，具体都可以见Django官方文档。


当一个key并不存在的时候，cache.get(key)会返回None。因为这样，官方强烈建议大家不要在cache里存值本身就为None的条目，因为这样拿到None的时候会混淆。


### Cache key
为了区分不同的环境或者服务器等（developping env, production env）可以给cache加个前缀 - settings里cache的配置加上KEY_PREFIX。 
每一个cache还可以有版本，比如
```python
cache.set('my_key', 'hello world!', version=2)
cache.get('my_key', version=2)
```

综合来说要得到一个cache完成的key，源码是
```python
def make_key(key, key_prefix, version):
    return ':'.join([key_prefix, str(version), key])
```
可以写自己自定义得到解析cache key的方法，比如把key做哈希

<br>

### Cache的配置有以下几种

## Memcached
memcached 的所有数据都会存在内存上，所以不会出现向数据库或文件系统溢出的问题。它会运行在后台（类似守护进程）并被分配一定量的ram（内存）
可以起多个instance并且配置：

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': [
            '172.19.26.240:11211',
            '172.19.26.242:11212',
            '172.19.26.244:11213',
        ]
    }
}
```

缺点是没有数据持久性，

## Database caching
```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.db.DatabaseCache',
        'LOCATION': 'my_cache_table',
    }
}
```
用数据库做cache的时候必须先

```bash
python manage.py createcachetable
```

表名就是location的值

## Filesystem caching
```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/var/tmp/django_cache',
    }
}
```
location should be an absolute path

## Local-memory caching
如果没有任何setting，这个cache是default的。 如果没有足够的性能去用memcached，可以选择这个。它的instance基于单个进程，并且是线程安全的。
```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'unique-snowflake',
    }
}
```
因为跨进程是不可能的，这种最好适用于开发环境，而不是生产环境

## Dummy caching
啥都不干。。。只是可以用所有的cache接口。专为了开发环境。

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}
```
<br>
## Settings中Middleware的顺序
最后说一下在settings中，如果要用cache的middleware，需要注意放置的顺序。


我们常看到settings中的middleware settings是类似这样：
```python
MIDDLEWARE_CLASSES = (
    'django.middleware.cache.UpdateCacheMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.auth.middleware.SessionAuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.middleware.cache.FetchFromCacheMiddleware',
)
```
首先要明确middleware的执行规则 - [官方文档1.11](https://docs.djangoproject.com/en/1.11/topics/http/middleware/)


Middleware是Django的request/response的钩子，在request请求中，执行到view之前，middleware的执行顺序是和代码中一样从上至下的。而在开始响应response的时候，middleware的执行是从下至上的（所谓的"the way back"）。如果某一层要短路的话，之后的层都不会接到任何request或response。官方文档中把middleware这种顺序比喻成一个洋葱从最外层到最内层再返回，如果一个钩子执行到了洋葱的某一层就往外返回了，洋葱里面的那些层是完全无感知的。


再看回cache相关的middleware，主要是django.middleware.cache.UpdateCacheMiddleware和django.middleware.cache.FetchFromCacheMiddleware。UpdateCacheMiddleware是在response层面执行，所以要确保它在一些会修改header的middleware之后运行，所以一般把它放最上面。而FetchFromCacheMiddleware是在request层面执行，它一样需要在可能更改header的middleware之后运行，所以可以把它放在最下面。