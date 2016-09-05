---
layout: post
title: web中的跨域请求--JSONP
excerpt: ""
date: 2016-09-05 09:23:32
permalink: /cross-domain-request-jsonp/
tags:
- JavaScript
- HTTP
---

## 同源策略

同源策略是一种约定,它是浏览器最核心也是最基本的安全功能,在1995年有Netscape公司引入浏览器, 目前,所有浏览器都实行这个政策。

所谓同源, 指的是:

>+ 协议相同
>+ 域名相同
>+ 端口相同

只有"三个相同"都被满足时, 才算符合同源。

目前而言, 如果非同源的话, 有三种行为会受到限制:

>+ Cookie, localStorage 和 IndexDB 无法读取。
>+ DOM无法获得。
>+ AJAX请求无法发送。

本文主要讲的是如何处理第三种情况, 分享三种方法:

>+ JSONP
>+ 服务器代理
>+ CORS

## JSONP

JSONP 是 `JSON with padding` 的缩写。核心思想就是将数据包含在一个回调函数中, 形成函数调用, 并在客户端执行。

由于同源策略对 script, img 这类的资源请求是无效的, 所以可以用 script 标签发起跨域请求。

浏览器端代码:

```javascript
function getJSONP(url, callback) {

	// 为每个JSONP 请求创建一个唯一的回调函数
	var cb = 'cb' + Date.now();
	getJSONP.cb = function(response) {
		callback(response);

		// 请求完成后清理掉创建的回调函数和script元素
		delete getJSONP.cb;
		script.parentNode.removeChild(script);
	}

	// 创建script元素用于发送请求
	var script = document.createElement('script');
	// 在script标签后指定回调函数
	script.src = url + (url.indexOf('?') == -1 ? '?' : '&') + 'callback=getJSONP.cb';
	// 通过将script元素加入到DOM中立即发送请求
	document.body.appendChild(script);
}

getJSONP('http://localhost:3000/api/bills', function(response) {
	console.log(response);
});
```

服务器端部分代码

```javascript
// 得到浏览器端传来的callback名
var urlInfo = parseUrl(req.url, true);
var callback = urlInfo.query.callback;
var data = {'foo': 'bar'};

// 如果有callback, 则拼装成函数调用的形式, 否则正常返回
if(callback) {
	res.end(callback + '(' + JSON.stringify(data) + ')');
} else {
	res.end(JSON.stringify(data));
}
```

这样当浏览器发出JSONP请求时, 返回的结果是

```javascript
getJSONP.cb1473088405312({
    'foo': 'bar'
})
```

因为它是一段 JavaScript 代码, 所以会被立刻执行, 调用我们刚刚定义的回调函数, 处理服务器返回的数据。

以上就是JSONP的原理, 其实很简单, 不要把它想的太复杂了...... JSONP 虽然简单易用, 但是也存在一些不足。

>+ 安全问题。如果请求的是不受信任的域,它很可能会在响应中夹杂一些恶意代码。因此在使用不是你自己运维的Web服务时, 就确保它是安全可靠的。
>+ 无法轻易的判端JSONP请求是否失败。
>+ JSONP 只能用于GET请求。