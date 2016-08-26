---
layout: post
title: Javascript 面向对象编程(1)
excerpt: "本文主要介绍了JavaScript面向对象编程中一些比较重要的概念--对象,类,原型"
date: 2016-08-23 09:11:34
permalink: /javascript-object-oriented/
tags:
- Javascript
---

## 概述

JavaScript的数据分为两类: 原始类型和对象类型。

原始类型有4种:

>+ 数值
>+ 字符串
>+ 布尔值
>+ Symbol(ES6新增)

Javascript还有两个特殊的值: `null`和`undefined`。它们是各自类型的唯一成员。

除了上述的原始类型,null,undefined以外,其他的数据类型都是对象了。

对象类型有可以分为三个子类型

>+ 狭义的对象(Object)
>+ 数组(Array)
>+ 函数(Function)

对象是一种复合值, 它是由很多值组合在一起形成的。它封装了属性和方法。属性代表了对象的状态,而方法代表了对象的行为。

```javascript
// 声明了一个dog对象,这个对象有name和age属性,并且有shout方法。
var cat = {
    name: 'Eddie',
    age: 3,
    shout: function() {
        console.log('wang~');
    }
}

cat.name        // 'Eddie'
cat.age         // 3
cat.greeting()  // wang~
```

> 小问题: 既然只有对象有属性和方法, 那么`'abc'.length`是怎么回事呢?为什么`'abc'`作为字符串也有length属性呢?

因为取字符串,数字或布尔值的属性时会自动创建一个临时对象,这是JavaScript的一个实现细节,被称为`"包装对象"`。

```javascript
var s = 'hello world';
s.length        // 11
```

在这个过程中,其实近似有一个这样的过程

```javascript
var s = 'hello world';
// 创建一个临时对象
tmp = new String(s);
// 由这个临时对象来处理属性的引用
tmp.length;
// 引用结束后,销毁新创建的临时对象
delete tmp;
```

每次引用结束后,临时对象都会被销毁,所以,给它定义新的属性是无效的。

```javascript
var s = 'hello world'
s.len = 3;
var l = s.len   // undefined
```

第二行创建了一个临时对象,并给它定义了一个len属性。但是第二行结束时临时对象就被销毁了。第三行试图引用len属性时又创建了一个新的临时对象,
而这个对象是没有len属性的,所以返回undefined。

## 类

在面向对象的思想中,有一个重要的概念是类。类是具有相同特性和行为的对象的抽象。类是对象的抽象,而对象是类的实例。比如, 狗是一个类,而蜡笔小新中的小白
则是一个对象,他是狗的一个实例。

在JavaScript中,是通过`构造函数`和`原型对象`来定义类的。

>+ 构造函数是一个专门用于生成对象的函数,它提供模板,描述对象的基本结构。随后,可以通过`new`操作符和构造函数来创建一组具有相同结构的对象。
>+ JavaScript的每个对象都继承另一个对象,后者称为"原型"(prototype)对象。原型对象上的所有属性和方法,都能被派生对象共享。
>+ 每一个构造函数都有一个prototype属性,这个属性就是实例对象的原型对象。

```javascript
function Dog(name) {
    this.name = name;
    this.shout = function() {
        console.log('wang~');
    }
}

Dog.prototype = {
    sleep: function() {
        console.log('zzzz~');
    }
}
```

上述代码中,Dog就是一个构造函数,为了和普通函数区分,构造函数的第一个字母通常大写。定义了构造函数后,你可以用`new`操作符来生成对象。Dog.prototype
就是Dog的实例对象的原型对象,所以Dog的实例对象都拥有Dog.prototype中定义的属性和方法。

```javascript
// 生成一个Dog的实例对象
var eddie = new Dog('Eddie');
eddie.name      // 'Eddie'
eddie.shout()   // 'wang~'
eddie.sleep()   // 'zzzz~'
```

## new

new操作符用于执行构造函数,返回一个实例对象。

```javascript
var eddie = new Dog('eddie');
```

上述命令执行后,发生了如下操作:

>+ new操作符创建了一个新的通用对象
>+ 将新对象的__proto__属性设置为Dog.prototype。
>+ 将该新对象作为this的值传递给Dog构造函数。
>+ 当从构造器返回时,JavaScrip将新对象赋值给eddie变量

即:

```javascript
var o = new Object();
o.__proto__ = Dog.prototype;
Dog.call(this,);
var eddie = o;
```

![object](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-08-26%20at%2011.58.26%20PM.png)

## prototype

既然有了构造函数,为什么还需要原型对象呢?构造函数不也能描述对象的模板,提供属性和方法吗?

```javascript
function Dog(name) {
    this.name = name;
    this.shout = function() {
        console.log('wang~');
    }
}

var eddie = new Eddie('Eddie');
var hobo = new Hobo('Hobo');

hobo.shout === eddie.shout          // false
```

在构造函数内部定义的属性和方法,所有实例对象都会生成。也就是说,每新建一个实例对象,都会新建一个name属性和shout方法。由上面那段可知,hobo的shout和eddie的shout并不是同一个方法。
这样做是对系统资源的浪费,因为同一个构造函数的对象实例之间,无法共享属性和方法。

所以需要prototype来帮忙。

上文提到,JavaScript的每个对象都继承自另一个对象,这个对象叫做"原型"对象。原型对象上的所有属性和方法,都能被派生对象所共享。

```javascript
function Dog(name) {
    this.name = name;
}

Dog.prototype = {
    shout: function() {
        console.log("wang~");
    }
}

var hobo = new Dog('hobo');
var eddie = new Dog('eddie');

hobo.shout === eddie.shout          // true
```

上述代码中,Dog.prototype就是eddie和hobo的原型对象。在原型对象上定义的属性和方法,是所有派生对象共享的。

```javascript
Animal.prototype.color = "yellow";
eddie.color         // 'yellow'
hobo.color          // 'yellow'
```