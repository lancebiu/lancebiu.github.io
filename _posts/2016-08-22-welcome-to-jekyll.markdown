---
layout: post
title:  Using ECharts with Angular.js
excerpt: "[ECharts](http://echarts.baidu.com) is a powerful JavaScript library to make amazing charts. This post introduces how to use ECharts in [*MVW*](http://stackoverflow.com/questions/13329485/mvw-what-does-it-stand-for) style."
date:   2016-08-22 00:22:54
permalink: /using-echarts-with-angularjs/
tags:
- AngularJs
- ECharts
---

> [ECharts](http://echarts.baidu.com) is a powerful JavaScript library to make amazing charts. This post introduces how to use ECharts in [*MVW*](http://stackoverflow.com/questions/13329485/mvw-what-does-it-stand-for) style.

## ECharts

ECharts uses a configurable `option` object to control most of its data and visual settings. A typical way to use ECharts after including `echarts.js` in HTML is as follows.

```js
var dom = document.getElementById('intro-chart');
var chart = echarts.init(dom);

chart.setOption({
    backgroundColor: '#08263a',
    title: {
        text: 'ECharts Example',
        textStyle: {
            color: '#b1cfa5',
            fontSize: 18
        },
        left: 'center',
        top: 25
    },
    xAxis: {
        show: false,
        data: ...
    },
    ...
    series: [{
        type: 'bar',
        data: ...,
        itemStyle: {
            normal: {
                barBorderRadius: 5,
                shadowBlur: 10,
                shadowColor: '#111'
            }
        },
        ...
    }]
});
```

To use ECharts with Angular, a very basic requirement is to bind chart option with ECharts instance.

## HTML

Suppose we have an HTML containing multiple elements for ECharts instances.

```html
<div class="dash-chart" eoption="vm.eoption.a"></div>
<div class="dash-chart" eoption="vm.eoption.b"></div>
<div class="dash-chart" eoption="vm.eoption.c"></div>
```

Here, `eoption` is a user-defined attribute, which could be anything you like.

## Controller

In Angular controller, we define options as follows. To make the demo simple enough, we only set a title in option.

```js
function MyController() {
    var vm = this;
    vm.eoption = {
        a: {
            title: {
                text: 'This is a'
            }
        },
        b: {
            title: {
                text: 'This is b'
            }
        },
        c: {
            title: {
                text: 'This is c'
            }
        }
    };
}
```

## Directive

In Angular directive, we watch each charts and updates option when attribute changes.

```js
function dashChart() {
    return {
        restrict: 'C',
        link: function (scope, elem, attrs) {
            // directive is called once for each chart
            var myChart = echarts.init(elem[0]);

            // listen to option changes
            if (attrs.eoption) {
                scope.$watch(attrs['eoption'], function() {
                    var option = scope.$eval(attrs.eoption);
                    if (angular.isObject(option)) {
                        myChart.setOption(option);
                    }
                }, true); // deep watch
            }
        }
    }
}
```

Note that we set the third parameter of `scope.$watch` to be `true`, which makes sure we get notified when any descendant of `eoption` changes.

To watch the change event of DOM attribute, a more straightforward way may be using Angular's `attrs.$observe`. Keller used `attrs.$observe` to watch DOM atrribute[^keller], but deep watching is not enabled in his example.

`attrs.$observe` doesn't support the third parameter stating if enables deep watching. **This means it will be triggered only when you set `vm.eoption.a` to a new value, but not when `vm.eoption.a.title` changes.**[^watch-vs-observe]

Considering changing part of option is a very common need, and ECharts suggests calling `setOption()` with minimal changed option, using `scope.$watch` seems to be a better choice over `attrs.$observe`.

## Changing Data

To demonstrate data-binding effect, I set the title to be current time, and update it every second.

```javascript
function MyController($interval, dateFilter) {
    var vm = this;

    vm.eoption = {
        a: {
            title: {
                text: ''
            }
        }
    };

    // update data every second
    $interval(function() {
        vm.echartsOption.userSessionChart.title.text =
            dateFilter(new Date(), 'yyyy-d-M HH:mm:ss');
    }, 1000);
}
```

You should see the Canvas is updating with current time.

[^keller]: [使用 angular 封装 echarts](http://www.kellerblog.cc/angular-echarts.html)
[^watch-vs-observe]: [AngularJS: Difference between the $observe and $watch methods](http://stackoverflow.com/questions/14876112/angularjs-difference-between-the-observe-and-watch-methods)
