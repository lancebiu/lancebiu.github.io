---
layout: post
title: web中的跨域请求--CORS
excerpt: ""
date: 2016-09-06 12:12:23
permalink: /cross-domain-request-cors/
tags:
- JavaScript
- HTTP
---

## CORS

CORS 是 Cross-Origin Resource Sharing 的缩写, 全称是"跨域资源共享"。

它是一个W3C标准, 它允许浏览器向跨域服务器, 发出XMLHttpRequest请求, 从而克服了AJAX只能同源使用的限制。

CORS 需要浏览器和服务器同时支持。CORS 标准通过新增一系列HTTP头, 让服务器能声明哪些来源可以通过浏览器访问该服务器上的资源。在浏览器端整个
过程是自动完成的, 不需要用户参与, 和同源AJAX通信的代码几乎一样。浏览器一旦发现AJAX请求跨域, 就会自动添加一些附加的头信息, 有时还会多出一次
附加的请求, 但这些对用户是透明的。

CORS 的浏览器支持情况如下图所示

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-09-05%20at%2011.33.45%20PM.png)

Chrome, Firefox, Opera 和 Safari 都是用 `XMLHttpRequest2` 实现的, 但是IE8, IE9 并不支持, 它引入了`XDomainRequest`对象, 这个对
象和XHR类似, 但是额外加入了一些[安全措施](https://blogs.msdn.microsoft.com/ieinternals/2010/05/13/xdomainrequest-restrictions-limitations-and-workarounds/)。

>+ 只支持 GET 和 POST 请求。
>+ 不能添加自定义请求头。
>+ 头部信息中的 Content-Type 只能设置为 text/plain
>+ cookie 不会随请求发送, 也不会随响应返回。
>+ 跨域请求的协议必须相同。如果你的地址为 http://example.com, 那么跨域请求的url 也必须是 http 协议的。如果你的地址为 https://example.com, 那么跨域请求的url 也必须是 https 协议的。


CORS 请求分为两种: 简单请求和非简单请求。

只有同时满足下列条件的请求才是简单请求。

(1) 请求方法是下列三个方法之一

>+ HEAD
>+ GET
>+ POST

(2) 请求头信息不超过以下字段

>+ Accept
>+ Accept-Language
>+ Content-Language
>+ Last-Event-ID
>+ Content-Type, 但仅限于三个值 application/x-www-form-urlencoded, multipart/form-data, text/plain

任何不满足上述的请求都是非简单请求, 简单请求和非简单请求的处理过程是不同的。

![](http://ocd7f3wcw.bkt.clouddn.com/cors-flow.png)

如上图所示, 非简单的 CORS  请求, 会在正式请求之前, 增加一次HTTP查询请求, 称为"预检"请求 (preflight)。

### 创建请求对象

我们先定义一个请求对象

```javascript
function createCORSRequest(method, url) {
	var xhr = new XMLHttpRequest();

	if ("withCredentials" in xhr) {
	    // withCredentials 只在XMLHttpRequest2 中存在
		xhr.open(method, url, true);
	} else if (typeof XDomainRequest != "undefined") {
	    // 检查是否有 XDomainRequest
		xhr = new XDomainRequest();
		xhr.open(method, url);
	} else {
	    // 当前浏览器不支持CORS
		xhr = null;
	}
	return xhr;
}
```

### 简单请求

我们先来发送一个简单请求。

```javascript
var url = 'http://jsonplaceholder.typicode.com/posts/1';
var xhr = createCORSRequest('GET', url);
xhr.send();
```

查看HTTP Request:

```http
GET /posts/1 HTTP/1.1
Host: jsonplaceholder.typicode.com
Origin: http://localhost:63342
User-Agent: Mozilla/5.0...
...
```

通过查看HTTP Request 我们可以发现请求头中添加了一个`Origin`字段。这个字段用于说明本次请求来自哪个源(协议 + 域名 + 端口)。服务器根据这个
值, 决定是否同意这次请求。

查看HTTP Response Headers:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:63342
Access-Control-Allow-Credentials: true
...
```

可以看到响应头中有一些以`Access-Control-`开头的字段, 这些响应头就是与 CORS 请求相关的, 我们来看看这些字段的含义。

>+ `Access-Control-Allow-Origin`(必须) - 这个响应头是必须要有的。如果没有这个响应头, 那个 CORS 请求就会失败。它的值要么
是请求时请求头`Origin`的值, 要么是`*`表示接受任意域名的请求。
>+ `Access-Control-Allow-Credentials`(可选) - 这个响应头是可选的, 通常情况下它并不包含在响应中, 它表示是否允许发送cookie。它只有
一个有效值`true`, 所以如果你不需要cookie, 不要添加这个响应头(而不是把它设为false)。
>+ `Access-Control-Expose-Headers`(可选) - XMLHttpRequest2 有一个 getResponseHeader() 方法用于返回指定的响应头。在 CORS 请求中,
这个方法只能访问6个基本的响应头: `Cache-Control`, `Content-Language`, `Content-Type`, `Expires`, `Last-Modified`, 'Pragma'。
如果想要访问其他的响应头, 就需要在`Access-Control-Expose-Headers`中指定。

### 非简单请求

简单请求并不能满足我们的所有需求, 我们有时需要使用`PUT`, `DELETE`方法, 又或是我们想要设置`Content-Type`为`application/json`,
这个时间我们的请求就是非简单请求。

非简单请求和简单请求的代码几乎是一样的, 但是非简单请求其实包含了两次请求。浏览器会先发送一个`预检`请求(preflight request), 这个请求
用于向服务器询问将要发起的请求是否被允许。如果请求被允许, 那个浏览器就会发出真正的请求, 否则就会报错。`预检`请求的响应是可以被缓存的。
这个过程对用户是透明的, 你无需手动处理两次请求, 浏览器会自动完成这一切。

我们来发起一个非简单请求。

```javascript
var url = 'http://jsonplaceholder.typicode.com/posts/1';
var xhr = createCORSRequest('PUT', url);
xhr.setRequestHeader('X-Custom-Header', 'value');
xhr.send({
	id: 1,
	title: 'foo',
	body: 'bar',
	userId: 1
});
```

打开调试工具, 可以观察到发送了两次请求, 先来看第一次请求。

```http
OPTIONS /posts/1 HTTP/1.1
Host: jsonplaceholder.typicode.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: x-custom-header
Origin: http://localhost:63342
User-Agent: Mozilla/5.0....
...
```

就和简单请求一样, 请求中同样有一个`Origin`请求头。预检请求的方法是`OPTIONS`, 它还包括一些其他的请求头:

>+ Access-Control-Request-Method - 说明了实际请求的HTTP方法。
>+ Access-Control-Request-Headers - 该字段是一个逗号分隔的字符串, 说明实际请求会额外发送的请求头。

预检请求就是在实际请求之前先询求服务器同意的, 服务器通过预检请求中的上述两个请求头来判断实际请求是否能被接受, 如果HTTP方法和请求头
被认定是可接受的, 那么服务器会作出如下响应。

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:63342
Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE
Access-Control-Allow-Headers: x-custom-header
Access-Control-Allow-Credentials: true
...
```

上面的响应中, 三个以`Access-Control-Allow-`开头的相应头是关键, 它们分别表示:

>+ `Access-Control-Allow-Origin`(必须) - 就像简单请求一样, 这个响应头是必须的。
>+ `Access-Control-Allow-Methods`(必须) - 这个响应头是必须的。它是用逗号分隔的字符串, 表明服务器支持的`所有`HTTP方法。
>+ `Access-Control-Allow-Headers` - 如果请求中包含`Access-Control-Request-Headers`头, 那么`Access-Control-Allow-Headers`就
是必须的。它是用逗号分隔的字符串, 表明服务器支持的`所有`头信息字段。
>+ `Access-Control-Allow-Credentials`(可选) - 就像简单请求一样, 这个响应头是可选的。
>+ `Access-Control-Max-Age`(可选) - 每次非简单请求都要先发出一个"预检"请求会对性能造成负担, 这个响应头可以用来指定本次"预检"请求
的有效期, 单位为秒。在此期间, 非简单请求将不再发送"预检"请求, 而是使用本次请求的结果。

如果服务器通过了"预检"请求, 那么浏览器就会发送真正的请求, 第二个请求就和简单请求类似。

查看第二次请求:

```http
PUT /posts/1 HTTP/1.1
Host: jsonplaceholder.typicode.com
Origin: http://localhost:63342
X-Custom-Header: value
User-Agent: Mozilla...
...
```

第二次请求的响应:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:63342
Access-Control-Allow-Credentials: true
...
```

如果服务器没通过'预检'请求, 会返回一个正常的HTTP回应, 但是没有任何 CORS 相关的头信息字段。这时, 会触发`xhr.onerror` 事件处理函数, 并在
控制台报错。

下面是 CORS 请求的流程图, 较为清晰的解释了 CORS 的整个流程

![](http://www.html5rocks.com/static/images/cors_server_flowchart.png)

### 服务器端实现

对于简单的 CORS 请求, 服务器只需要在响应头中添加`Access-Control-Allow-Origin: *` 即可, 如果有更多需求, 比如安全性等等, 可以在响应头
中做更多配置。 这里有[一份列表](http://enable-cors.org/server.html), 列举了常见的服务器如何果支持 CORS。


## CORS 和 JSONP 的比较

在[上一篇文章](http://devdeeper.com/cross-domain-request-jsonp/)中, 提到了 JSONP 的原理和它的一些不足。和 CORS 相比, JSONP 只支持`GET`请求, 而 CORS 支持所有类型的HTTP请求,
但是 JSONP 可以支持所有浏览器, 而 CORS 在一些老式浏览器上并不支持。

>+ 如果仅需要只读的跨域请求并且需要兼容低版本浏览器, 就使用JSONP。
>+ 如果需要支持多种HTTP方法(比如使用REST API)或要在请求中添加一些头信息时, 使用 CORS。