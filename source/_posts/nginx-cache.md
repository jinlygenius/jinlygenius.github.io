---
title: Nginx Cache
date: 2017-09-08 15:50:49
tags:
- cache
- nginx
---

也是先放参考文章镇楼

[http://www.oschina.net/question/54100_29363](http://www.oschina.net/question/54100_29363)

[http://seanlook.com/2015/05/17/nginx-install-and-config/](http://seanlook.com/2015/05/17/nginx-install-and-config/)


然后来聊一下最近我们遇到的问题。一直用django设置缓存，发现无论怎么改变max-age, 最后正式环境服务器总是输出72小时缓存。后来老大意识到是nginx的缓存覆盖了django我们设置的。

解决方法暂时是去掉了nginx的缓存，完全交给django控制。

```
location / {
    proxy_pass xxx;
    proxy_redirect default;
    expires off;
}
```

加上expires off; 即可。 