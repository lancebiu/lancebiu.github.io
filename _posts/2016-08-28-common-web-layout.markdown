---
layout: post
title: 自适应三栏布局的实现
excerpt: "三栏布局是web中常见的一种布局方式, 通常的要求是要实现页面自适应, 不管在实际应用还是面试中都是经常遇到的,本文介绍了我自己掌握并经常使用的三种方法。"
date: 2016-08-27 00:17:24
permalink: /three-column-layout/
tags:
- CSS
- 布局
---

## 前言

三栏布局是web中常见的一种布局方式, 通常的要求是要实现页面自适应, 不管在实际应用还是面试中都是经常遇到的。

下面介绍了我自己掌握并经常使用的三种方法,以实现: 左中右三栏布局, 左右栏宽200像素, 中间栏宽度自适应。

(1)(2)(3) 代表在HTML结构中出现的先后顺序。

### 方法1:

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-08-28%20at%202.13.48%20PM.png)

[Demo传送门](http://codepen.io/lancebiu/pen/RRXLEE)

### 方法2:

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-08-28%20at%202.13.15%20PM.png)

这个方法要注意的是中间的主体要使用双层标签。外层宽度100%显示,并且浮动,内层为真正的主体内容。

[Demo传送门](http://codepen.io/lancebiu/pen/EyqwBo)

### 方法3:

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-08-28%20at%202.52.05%20PM.png)

这个方法用了Flexbox, 想了解更多关于Flexbox的知识,请戳[这篇文章](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)。

[Demo传送门](http://codepen.io/lancebiu/pen/GqVONK)