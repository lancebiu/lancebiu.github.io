---
layout: post
title: Javascript 面向对象编程(2)
excerpt: "本文主要介绍了JavaScript面向对象编程中类的继承和原型链相关的知识。"
date: 2016-08-26 11:32:34
permalink: /javascript-object-oriented-2/
tags:
- JavaScript
---

## 原型链

在[上一篇文章](http://devdeeper.com/javascript-object-oriented-1/)中, 我们说过创建一个对象的过程。在JavaScript中,每一个对象都有一个原型(prototype)对象,
对象会拥有其原型对象的所有属性和方法,可以通过`__proto__`查看对象的原型对象。原型对象同样也有自己的原型对象,直到某个对象的原型对象为null为止。这种一级一级的链式结构就叫做原型链。

```javascript
var a = {
    name: 'lance',
    age: 21
};

var b = {
    school: 'NJU',
    city: 'Nanjing'
}

a.__proto__ = b;
```

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-08-27%20at%208.53.47%20PM.png)

如上所示,a的原型对象是b, b的原型对象是`Object.prototype`, Object.prototype的原型对象就是null了, 所以整个原型链是
`a ---> b ---> Object.prototype ---> null`

当访问一个对象的属性时,就会按照原型链的顺序层层向上搜索,直到找到一个名字匹配的属性或达到原型链的末尾。

```javascript
// 按照 a ---> b ---> Object.prototype ---> null 的顺序查找
// 找到了则返回相应属性,若只到原型链的末尾还没找到则返回undefined
a.name              // a √      找到了,返回 'lance'
a.school            // a × ---> b √      找到了,返回 'NJU'
a.toString()        // a × ---> b × ---> Object.prototype √    找到了,返回[object object]
a.title             // a × ---> b × ---> Object.prototype × ---> null  直到原型链末尾还没找到, 返回undefined
```

## 类的继承

了解了原型链之后, 就能很容易的理解JavaScript是怎么实现继承的了。

正是因为JavaScript基于原型的这个特性, 所以JavaScript的继承也是基于原型的继承。

```javascript
function Person(name, age) {
    this.name = name;
    this.age = age;
}

Person.prototype.say = function() {
    console.log('hello, my name is ' + this.name);
}

function Man(name, age) {
    Person.apply(this, arguments);
}

Man.prototype = Object.create(Person.prototype);
Man.prototype.constructor = Man;

var mike = new Man('Mike', 18);

mike.name           // 'Mike'
mike.age            // 18
mike.say()          // 'hello, my name is mike'
```

上面的代码就是实现JavaScript继承的一种很普遍的方法。

第一步是在子类的构造函数中,调用父类的构造函数。

```javascript
function Man(name, age) {
    Person.apply(this, arguments);
}
```

这样就会让子类实例具有父类实例的属性。

第二步就是让子类的原型指向父类的原型。

```javascript
Man.prototype = Object.create(Person.prototype);
Man.prototype.constructor = Man;
```

这样做之后我们新建一个子类的实例对象来看看它的原型链。

```javascript
var mike = new Man('mike', 18);
```

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-08-27%20at%2010.32.18%20PM.png)

原型链为 `Man.prototype ---> Person.prototype ---> Object.prototype ---> null`

可以看到,mike的原型链中有Person.prototype,所以mike也就拥有了Person这个类的属性和方法。
