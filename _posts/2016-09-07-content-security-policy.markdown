---
layout: post
title: 【译】Content Security Policy 介绍
excerpt: ""
date: 2016-09-07 06:12:14
permalink: /content-security-policy/
tags:
- HTTP
---

> 本文翻译自 [html5rocks](http://www.html5rocks.com/) 的一篇文章, 原文链接 [An Introduction to Content Security Policy](http://www.html5rocks.com/en/tutorials/security/content-security-policy)。
翻译水平有限, 仅供参考。

***

Web安全模型是建立在[同源策略](https://en.wikipedia.org/wiki/Same_origin_policy)的基础上的, https://mybank.com 上的代码
只能访问 https://mybank.com 上的数据, 而 https://evil.example.com 则不能访问 https://mybank.com 上的数据。 每个域都与web上
其他域是隔离的, 这给开发者营造一个用于开发使用的安全沙盒。理论上来说这个策略是非常聪明的做法, 但是实际上, 攻击者发现了各种高招来推翻
了这套保护。

比如, XSS 攻击者通过将恶意代码注入到网站常规数据里来绕过同源策略。这是个大问题, 因为浏览器相信来自安全域的所有代码。XSS Cheat Sheet
是一种古老但是很有代表性的攻击方式。 一旦攻击者成功地注入了代码, 那么基本就 Game over 了: 用户数据暴露无遗, 一些需要保密的信息被泄露
给了不怀好意的人。我们迫切希望能够阻止这种攻击的发生。

本教程指出了一种全新的防御方式: Content Security Policy(CSP)。CSP 可以有效降低现代浏览器下 XSS 攻击的风险。

## 白名单

XSS 攻击主要抓住了浏览器无法区分脚本是来自自己的应用还是来自第三方攻击者的恶意注入这个缺陷。例如: Google的 +1 按钮会在本域中加载执行一个
来自 https://apis.google.com/js/plusone.js 的脚本。我们信任该脚本, 但是我们不能指望浏览器去区分来自 apis.google.com 的脚本是安全的,
而来自 apis.evil.example.com 就不安全。浏览器只会傻傻地下载和执行页面中的所有代码, 而不去管这段代码来自何方。

CSP 定义了`Content-Security-Policy` HTTP 头来允许你创建一个你信任的来源的白名单, 浏览器只允许加载来自白名单中各个域的资源, 执行来自白名单中
各个域的代码, 而不是盲目的信任任何来源。这样一来, 就算攻击者找到了一个漏洞来注入脚本, 但是由于该脚本的来源不在白名单中, 该脚本照样无法执行。

因为我们信任来自 api.google.com 和我们自己的代码, 所以我们可以这样定义我们的协议:

```
Content-Security-Policy: script-src 'self' https://apis.google.com
```

很简单吧? 顾名思义, `script-src` 控制了指定页面script标签相关的策略。我们指定了`'self'`和`https://apis.google.com`作为script标签
的有效来源, 这样浏览器就只会下载和执行本域还有 https://apis.google.com 的JavaScript脚本。

如果有来自其他域的脚本, 浏览器会抛出如下异常。就算攻击者成功地把脚本注入到了网站中, 但是等待他的也只是无情的报错。

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-09-07%20at%203.24.24%20PM.png)

## 策略支持多种资源

除了 script 资源这种最明显的安全风险外, CSP还提供了一系列指令来控制页面中多种资源的加载。你已经看到了`scrip-src`, 我们快速过一下
剩下的资源指令。

>+ `base-uri` 限制了页面中 \<base\> 元素中的 URL。
>+ `child-src` 列举了 workers 和 嵌入的 frame 内容的 URL。例如: `child-src https://youtube.com` 将只允许你嵌入来自youtube的
视频, 而不允许嵌入来自其他域的内容。
>+ `connect-src` 限制了列举了可以连接的源(通过XHR, WebSockets 和 EventSource)。
>+ `font-src` 指明了 web fonts 的可用来源
>+ `form-action` 列举了提交\<form\>表单的有效端点。
>+ `frame-ancestors` 指定可以嵌入当前页面的来源。这个指令适用于\<frame\>, \<iframe\>, \<embed\>和\<applet\>标签, 这个指令
不能在\<meta\>标签中使用, 并且只使用与非html资源。
>+ `frame-src` 启用。使用`child-src`。
>+ `img-src` 定义了图像可以从何处被加载
>+ `media-src` 限制了允许提供视频和音频的源
>+ `object-src` 运行控制flash和其他插件。
>+ `plugin-types` 限制了页面可以调用的插件的种类。
>+ `report-uri` 指定一个URL, 当某个安全策略被违反时, 浏览器会发送报告给指定的URL。这个指令不能在\<meta\>中使用。
>+ `style-src` 和`script-src`作用差不多, 只不多这个针对的是样式表
>+ `upgrade-insecure-requests` 指示用户代理重写URL协议,把 HTTP 转变成 HTTPS。这个指令是在网站有大量旧的 url 需要重写时使用。

默认情况下, 如果你不指定这些指令的值, 那么就允许所有的来源, 来自任何域的资源或脚本都能被加载或执行。例如 `font-src:*` 和不写 `font-src`
效果是一样的。

当然你也可以通过`default-src`指令重写默认值。这个指令定义了你未定义的大多数指令的默认值。通过, 这个对以`-src`结尾的指令有效。例如, 如果
指定了`default-src: https//example.com`, 但是没有指定`font-src`, 那么你的 font 也只能从 http://example.com 获取。在之前的例子中
我们只指定了`script-src`, 这说明我们可以从其他任何来源加载 image, font 和其他资源。

下列指令不受`default-src`影响。如果没有设置它们的话默认允许任何来源。

+ base-uri
+ form-action
+ frame-ancestors
+ plugin-types
+ report-uri
+ sandbox

你可以为你的web应用指定一个或多个指令, 只需要在 HTTP 头里面列出来这些值, 不同的值用`';'`分隔。在一个指令中你需要列出所有你需要的来源。
如果像下面这种写法:

```
script-src https://host1.com; script-src https://host2.com
```

第二个指令会被忽略, 下面的写法才会指定多个来源:

```
script-src https://host1.com https://host2.com
```

再举一个例子, 如果你的静态资源全部来自一个CDN, 比如说 https://cdn.example.net, 并且你知道你不需要frame和其他插件, 那么你可以这样写:

```
Content-Security-Policy: default-src https://cdn.example.net; child-src 'none'; object-src 'none'
```

## 实现细节

在其他很多教程中会出现`X-Webkit-CSP`和`X-Content-Policy`头。 但是我们应该向前看了, 我们能够并且应该忽略那些带前缀的HTTP头。
现代浏览器(除了IE)都已经支持不带前缀的`Content-Security-Policy`头了, 这才是我们该使用的。

CSP 策略要求每一个你希望得到保护的页面请求的响应都要带上`Content-Security-Policy`头, 这提供了很多灵活性, 你甚至可以根据条件判断什么时候
需要采用什么 CSP 策略。

每个指令的 source list 也是很灵活的, 你可以指定协议(data:, https:), 或者只指定主机名(example.com, 这样就允许该主机的任何协议, 任何端口),
或者指定一个完整的URI(https://example.com:433, 这个只匹配来自example.com的433端口, 并且使用https协议的源)。甚至可以使用通配符, 但是只能用于
子域, 端口和协议, 比如 *://*.example.com:* 将匹配example.com 的所有子域, 任意协议或端口。

下面四个关键词也可以使用在 source list 中:

>+ 'none' : 什么也不匹配
>+ 'self' : 匹配当前域, 不包括子域
>+ 'unsafe-inline' : 允许 inline JavaScript 和 CSS
>+ 'unsafe-eval' : 允许 eval, new Function 等执行。

这几个关键字都要用单引号引用起来。如果不引用, 则会被认为是域名。例如`script-src 'self'`表示当前域的JavaScript可以被加载执行, 而`script-src self`
则表示来自域 'self' 的JavaScript可以被加载执行, 这就产生了偏差。

## meta 标签

CSP 希望通过HTTP头的形式指定 CSP 策略。但是也可以直接在在页面标记中指定要使用的策略, 可以通过`http-equiv`属性实现:

```html
<meta http-equiv="Content-Security-Policy" content="default-src https://cdn.example.net; child-src 'none'; object-src 'none'">
```

这个方法对`frame-ancestors`, `report-uri` 和`sandbox` 是无效的。

## 内联代码有风险

CSP 是基于白名单的, 这种方式可以很清楚的区分哪些资源能被认可, 哪些不能。但是这种方法没能解决 XSS 攻击最大的一个问题: 内联脚本注入。
如果攻击者可以注入一段恶意脚本:

```javascript
<script>sendMyDataToEvilDotCom()</script>
```

浏览器没法区分这是不是一段恶意脚本。CSP 解决这个问题的方式就是`禁止所有内联脚本`。

不仅仅是`<script>`元素中的脚本, 还包括内联事件和`javascript: URL`, 它们都被禁止了。你需要把`script`元素中的内容都放在一个外部
脚本中, 并且把`javascript: URL`和`<a ... onclick="[JAVASCRIPT]">` 这样的代码都用`addEventListener`重写。例如, 下面这段代码:

```html
<script>
  function doAmazingThings() {
    alert('YOU AM AMAZING!');
  }
</script>
<button onclick='doAmazingThings();'>Am I amazing?</button>
```

就被重写为:

```html
<!-- amazing.html -->
<script src='amazing.js'></script>
<button id='amazing'>Am I amazing?</button>
```

```javascript
// amazing.js
function doAmazingThings() {
  alert('YOU AM AMAZING!');
}
document.addEventListener('DOMContentReady', function () {
  document.getElementById('amazing')
          .addEventListener('click', doAmazingThings);
});
```

除了满足 CSP 外, 上述代码的重写还有很多好处。哪怕你不使用 CSP 上述写法也是一个很好的方式。内联脚本把结构和行为混在了一起。
外部资源能够被浏览器缓存, 更容易被开发者理解, 而且能更好的被编写和压缩。把代码写在外部脚本中能写出高质量的代码。

内联样式也是同样的处理方法: 所有`<style>`元素或是元素的style属性都应该被写在外部样式表中以避免内联CSS 的一些可能存在的风险。

如果你坚决需要内联脚本或内联样式, 你可以通过使用`unsafe-inline`作为`script-src`和`style-src`的值, 但我们不建议这样做。
禁止内联脚本和样式是CSP提供的最大的安全机制, 禁止内联脚本和内联样式让你的应用更加坚不可摧。将内联脚本和内联样式正确的移到外部脚本
或外部样式表中也许需要一些代价, 但是这是值得作出的妥协。

## 如果你真的必须要使用它...

CSP 2 提供了一种向后兼容的的方式来支持内联脚本。可以通过使用一个加密随机数或者hash来标记内联脚本并把它加入到白名单中。这虽然会显得很
累赘, 但是在必要时这种方法很有用。

你可以给你的内联脚本加上一个随机数属性:

```html
<script nonce=EDNnf03nceIOfn39fn3e9h3sdfa>
  // Some inline code I can't remove yet, but need to asap.
</script>
```

然后把该随机数加上`nonce-`添加到`script-src`指令中。

```html
Content-Security-Policy: script-src 'nonce-EDNnf03nceIOfn39fn3e9h3sdfa'
```

记住随机数每次请求时都要重新生成并且生成的结果要是不可预测的。

hash 的工作原理也大致相同。但是这种方式是对 script 元素本身创建一个 SHA hash 并且把计算出的 hash 添加到`script-src`指令中, 而不是
在 script 元素中添加额外的代码。例如, 页面中有如下内联脚本:

```html
<script>alert('Hello, world.');</script>
```

那么你的 CPS 策略就会包含:

```html
Content-Security-Policy: script-src 'sha256-qznLcsROx4GACP2dm0UCKCzCG-HiZ1guq6ZZDob_Tng='
```

有一些地方我们要注意一下。`sha*-`前缀指定了hash生成算法。上例中使用的是`sha256-`, CSP 还支持`sha384-`和`sha512-`. 生成hash使用的
是 script 元素内的内容, 不要带上\<script\>标签。 而且字母大小写和空格都是有影响的, 不要忘记开始或结束出的空格。

Google 搜索 SHA hash 生成算法, 它会指引你获得各种语言的生成方式。如果使用Chrome 40 或更新的版本你可以打开开发者工具并重新载入你的页面,
console 标签会有错误信息, 错误信息中包含了每个内联脚本正确的 sha256 hash。

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-09-07%20at%2011.03.01%20PM.png)

## Eval

尽管攻击者不能直接注入脚本, 他也可能会使用一些把字符串转换成可执行的 JavaScript 代码的方式进行攻击。比如eval(), new Function(),
setTimeout([string], ...) 和 setInterval([string], ...) 这些载体都有可能会导致一些恶意代码的执行。CSP 对这个问题的处理方式, 就是
完全禁止所有这些载体。

这一点会对你开发应用的方式造成一些改变。

+ 用内置的`JSON.parse`转化 JSON, 避免使用 eval。原生的 JSON 操作在IE8之后的所有浏览器上都是可用的, 并且是觉得安全的。
+ 重写所有使用 string 而不是内联函数的 setTimeout 和 setInterval 调用。

```javascript
setTimeout("document.querySelector('a').style.display = 'none';", 10);
```

&nbsp; &nbsp; &nbsp; &nbsp;重写为:

```javascript
setTimeout(function () {
    document.querySelector('a').style.display = 'none';
}, 10);
```

+ 避免运行时的内联模板。很多模板库都使用`new Function()`加速运行时模板的生成。 这种做法很棒, 但是也带来了执行恶意代码的风险。
有些库支持不带`new Function()`的 CSP 版本, 降级到不适用 eval 的版本。

如果你使用的模板支持预编译, 那就最好了。预编译模板将比最快的运行时编译模板都要快, 并且更安全。如果你必须在你的应用中使用这类
`text-to-javascript`方法, 加入`unsafe-eval`到`script-src`中, 但是, 最好不要这样。阻止字符串的运行可以让攻击者在你的网站
执行恶意代码变得更加困难。

## 报告

CSP 阻止不受信任的资源在客户端执行的能力对用户来说是巨大的胜利, 但是如果能传回服务器一些报告将更棒, 这样你就能第一时间定位bug, 找到
能够被注入恶意代码的地方。为此, 你可以通过在`report-uri`中指定一个地址告诉浏览器如果发生了攻击, 就把 JSON 格式的报告发送到指定的地址。

```
Content-Security-Policy: default-src 'self'; ...; report-uri /my_amazing_csp_report_parser;
```

报告内容是以下格式:

```json
{
  "csp-report": {
    "document-uri": "http://example.org/page.html",
    "referrer": "http://evil.example.com/",
    "blocked-uri": "http://evil.example.com/evil.js",
    "violated-directive": "script-src 'self' https://apis.google.com",
    "original-policy": "script-src 'self' https://apis.google.com; report-uri http://example.org/my_amazing_csp_report_parser"
  }
}
```

它包含很多能够帮助你找到攻击的源头的信息, 包括攻击发生的页面(document-uri), 页面的来源(referrer), 违反页面策略的资源(blocked-uri),
该资源违反的具体的指令(violated-directive), 还有页面完整的 CSP 策略(original-policy);

## 只报告

如果你刚开始使用 CSP, 在推出严苛的策略之前你可以先评估一下你的应用目前的状态。作为你完全部署前的垫脚石, 你可以要求浏览器按照某种策略进行
监视, 一旦发生了攻击就向服务器发送报告, 但是并不要阻止脚本和其他资源的加载与执行。为此, 你需要发送一个`Content-Security-Policy-Report-Only`
头而不是发送`Content-Security-Policy`。

```
Content-Security-Policy-Report-Only: default-src 'self'; ...; report-uri /my_amazing_csp_report_parser;
```

`report-only`模式下并不会阻止受限制的资源和脚本的加载和执行, 但是它会向服务器发送报告。你甚至可以同时发送这两个头信息,
对于某些资源作出限制, 而另一些资源则只需要在违反策略时发送报告。这是评估你网站 CSP 效果的很棒的方法: 在实现一个新的策略前
先让它只报告, 监测报告并修复发现的bug, 一旦你觉得不错了就强制执行这个新策略。

