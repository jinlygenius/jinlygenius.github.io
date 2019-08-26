---
title: CORS 跨域的深入理解 后端视角
date: 2019-08-23 10:48:32
tags:
- cors
---


我们现在的B端管理网站，域名是sparrow.xxx.com 。 后端接口是 backend.xxx.com。 一直以来后端服务器都是允许跨域的，实现方式是django框架 + 包插件django-cors-headers。 惊觉为什么这样还是安全的呢？！拉上前端小伙伴大家认真重新复习了一下跨域，又动手试了试怎么能攻击到 backend.xxx.com 【捂脸】

### 跨域攻击
跨域攻击的基本原理是，浏览器访问网站A，网站A登录信息等存在相应的cookie中（此处前端小伙伴说，cookie，storage等都是和域名相关的，只有访问同域名的时候才能获取到）。这时候用户在另一个页卡打开了网站Z，假如网站Z是一个恶意网站，植入了恶意js脚本，这个时候这段恶意脚本就可以带着篡改过的数据，访问网站A，并带上之前的cookie中A的登录信息等。如果A是银行网站，Z执行的恶意攻击可能就是访问A并把用户的钱转账给其他不法账户。

如果上述A网站服务器不允许跨域，Z的ajax请求访问A的时候，因为不同源（必须同样的协议比如https，同样的域名包括二级域名都要相同，同样的端口比如80），A服务器就不会响应Z的请求，也就保障了安全。

### 我产生的问题
目前我们项目的配置是，普通用户只能访问网站 sparrow.xxx.com（纯前端node应用） ，sparrow里ajax访问backend.xxx.com，登录backend的方式是token认证，然后token存在浏览器。sparrow服务器是不允许跨域的（后续大家讨论允许跨域也还是安全的），backend服务器是允许跨域的，返回header里 Access-Control-Allow-Origin: * . 这样为什么是安全的呢？

### 问题详解

#### 概念一 明白实际发生了什么
1. 用户浏览器输入sparrow网址，访问sparrow网站
2. 浏览器请求到sparrow的js，里面ajax访问backend进行登录
3. 获取到backend的token，存在local storage里（关联的是sparrow域名）
4. 假如用户打开恶意网站Z
5. 因为sparrow服务器禁止跨域，所以Z无法请求sparrow服务器。所以是安全的。
6. 如果Z想拿着sparrow的local storage里的token去访问backend，因为token的源是sparrow，所以浏览器限制了没办法拿着去访问backend。所以也是安全的。
7. 那么假设，sparrow服务器也设置了允许跨域。那么Z可以拿着token去请求sparrow服务器，但是因为token是backend的，所以请求sparrow没有任何意义。所以还是安全的。


**这里就不得不引入另一个现实情况了——backend.xxx.com 的后端实现是django+drf。也就是提供服务端渲染的页面，可以用session的方式登录。于是又脑洞大开，那如果在浏览器第一个tab里，先用session的方式登录backend，然后在另一个tab访问恶意网站Z，由于backend是允许跨域的，Z是不是就可以用cookie里的session访问backend了，是不是就会有问题了？**


#### 概念二 明白请求头header里发生了什么
1. Access-Control-Allow-Origin: \*。 我之前一直质疑写\*不是很不安全么，为什么不只写sparrow.xxx.com不是感觉更安全么。实际并不是这样。 当浏览器请求服务器，第一次option请求的时候，服务器返回response header Access-Control-Allow-Origin: \* ，只有当值是\*的时候，是不允许携带cookie的。回到上面假想的访问backend的时候用session的登录方式，会不会不安全？答案是安全的，因为Access-Control-Allow-Origin: \* 要不就同源访问，要是非同源的话是不允许带cookie的。

2. HttpOnly cookies。在第一次option访问backend的时候，response header里会指明 Set-Cookie: id=a3fWa; Expires=Wed, 21 Oct 2015 07:28:00 GMT; Secure; HttpOnly。然后这种cookie就被浏览器保护起来了，没有任何js可以直接访问这个cookie。此cookie只能在访问backend的时候被浏览器自己控制放入request header中。所以再一次保证了不被Z挟持。（这里如果想了解更多可以看看 XSS 跨域脚本攻击）

是不是感觉清楚多了，完全明白了现在的服务器允许跨域为什么是安全的了。

#
**小伙伴们又问了，为什么现在都流行用token比如JWT取代session的方式登录呢？**

### token VS session —— 
作为一个后端，我认为token对后端的改进是比较大的，因为如果用session的话，数据库里需要存session表，根据前端发来的session_id查表才能获取到用户。而token比如JWT，分发之后就不需要再储存了，校验的时候只需要用secret加盐校验是自己分发出去的有效token，再从token payload 取出用户id就可以了。而对前端来说，存session_id和token，都是两个字符串，感觉没什么区别。

不过我再一次naive了。实际上token在前端对比session_id还有另一个好处，就是顺便解决了csrf跨站请求伪造问题。当浏览器通过form表单提交服务器的时候，cookie是被自动带上的，且不遵循同源原则。也就是说，如果恶意网站Z的js通过form提交网站A的服务器，且篡改了提交数据，且登录A是通过session_id存cookie的方式，就可以达到文章开头提到的转钱给不法账户的效果。这个时候通常的解决方法是通过form携带csrf token，也就是表单每次提交的时候，都需要携带A服务器分发的一个csrf token，后端会校验csrf token是否有效。这样就能避免csrf攻击了，因为Z的js是无法自己获取到一个新的有效的csrf token的。那么，如果不是用session的登录方式，而是用token authorization的方式，表单提交的时候自动带上的cookie就没用了。原理其实就类似csrf token，请求需要把JWT token从local storage取出来放在请求header里。这样流程就回到了文章上述的各流程中，也就避免了csrf攻击。所以即使对前端来说，token登录方式也比session的登录方式更有优势。

#
### django-cors-headers
最后再看一下服务器允许跨域是怎么实现的。首先在settings.py 里配置 CORS_ORIGIN_ALLOW_ALL = True 表示允许跨域。然后查看代码 corsheaders/middleware.py 中 CorsMiddleware 中

```python
def process_response(self, request, response):
        """
        Add the respective CORS headers
        """
        enabled = getattr(request, "_cors_enabled", None)
        if enabled is None:
            enabled = self.is_enabled(request)

        if not enabled:
            return response

        patch_vary_headers(response, ["Origin"])

        origin = request.META.get("HTTP_ORIGIN")
        if not origin:
            return response

        # todo: check hostname from db instead
        url = urlparse(origin)

        if conf.CORS_ALLOW_CREDENTIALS:
            response[ACCESS_CONTROL_ALLOW_CREDENTIALS] = "true"

        if (
            not conf.CORS_ORIGIN_ALLOW_ALL
            and not self.origin_found_in_white_lists(origin, url)
            and not self.check_signal(request)
        ):
            return response

        if conf.CORS_ORIGIN_ALLOW_ALL and not conf.CORS_ALLOW_CREDENTIALS:
            response[ACCESS_CONTROL_ALLOW_ORIGIN] = "*"
        else:
            response[ACCESS_CONTROL_ALLOW_ORIGIN] = origin

        if len(conf.CORS_EXPOSE_HEADERS):
            response[ACCESS_CONTROL_EXPOSE_HEADERS] = ", ".join(
                conf.CORS_EXPOSE_HEADERS
            )

        if request.method == "OPTIONS":
            response[ACCESS_CONTROL_ALLOW_HEADERS] = ", ".join(conf.CORS_ALLOW_HEADERS)
            response[ACCESS_CONTROL_ALLOW_METHODS] = ", ".join(conf.CORS_ALLOW_METHODS)
            if conf.CORS_PREFLIGHT_MAX_AGE:
                response[ACCESS_CONTROL_MAX_AGE] = conf.CORS_PREFLIGHT_MAX_AGE

        return response
```

可以看到，如果设置允许跨域，response[ACCESS_CONTROL_ALLOW_ORIGIN] = "\*" 这段代码会设置header中的ACCESS_CONTROL_ALLOW_ORIGIN为\*，和我们实际查看的response header对应上了。