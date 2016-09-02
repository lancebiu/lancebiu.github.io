---
layout: post
title: React Koa 同构实践
excerpt: "最近因为在项目中使用了React并需要在服务器同构直出, 所以写下这篇博客把这个过程中的一些细节和遇到的问题整理一下并进行分享。"
date: 2016-08-28 08:15:24
permalink: /koa-react-server-side-render/
tags:
- Koa
- React
---

# 前言

最近因为在项目中使用了React并需要在服务器同构直出, 所以写下这篇博客把这个过程中的一些细节和遇到的问题整理一下并进行分享。本文将以一个简单的
web应用来进行讲解 —— 只有两个页面, 一个列表页和一个详情页, 点击列表页上的条目进入详情页。

[文中案例完整代码下载 >>](https://github.com/lancebiu/koa-react-redux-server-side-render)

## 什么是同构

在web开发领域,同构的意思是同时在服务器和客户端渲染页面。同构JS通常是通过 Node.js 实现的,因为它支持可以复用的第三方库,同时支持代码在只有少量
修改的情况下同时运行在浏览器和 Node.js 环境上。由于有着这样的特性, Node.js 和 JavaScript 生态有着一堆支持同构的框架,例如React.js。

![](http://7rf34y.com2.z0.glb.qiniucdn.com/c/0d5dfafd17fd9aaa3e6a1453cb92809c)

## 为什么要使用同构

>+ 更好的搜索引擎优化(SEO)
>+ 首屏优化, 更好的性能
>+ 更好的维护性

## React同构

这次同构, 我采用了React + Redux + React-Router + Webpack 的架构。

### Redux

[Redux](http://redux.js.org/) 提供了一个类似于Flux的单项数据流。

>+ 应用中所有的`state`都以一个对象树的形式存储在一个单一的`store`中。
>+ 唯一改变`state`的方法是触发`action`——一个描述发生什么的对象。
>+ 为了描述`action`如何改变`state`树,你需要编写`reducers`。

Redux大致的工作流程如下图。

![](http://ocd7f3wcw.bkt.clouddn.com/redux%20%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B0.png)

单个应用只维护一个 store, 因此对于整个应用来说, 一个 store 就是一个UI快照, 它保存了应用的所有状态。因此,服务器渲染变得简单了起来。只要在服务器初始化 store, 并在直出时把 store 的 state
输出到客户端, 在客户端使用该 state 初始化 store, 就完成了数据的传递和共用。

以列表页为例, 我们定义了两个 action creator 和 一个 thunk action creator。

```javascript
export const REQUEST_LATEST_BILLS = 'REQUEST_LATEST_BILLS';
export const RECEIVE_LATEST_BILLS = 'RECEIVE_LATEST_BILLS';


// action creator
function request_latest_bills() {
    return {
        type: REQUEST_LATEST_BILLS
    }
}

// action creator
function receive_latest_bills(bills) {
    return {
        type: RECEIVE_LATEST_BILLS,
        bills: bills.items
    }
}

// thunk action creator
export function fetchLatestBills() {
    return dispatch => {
        dispatch(request_latest_bills());
        return fetch(`http://localhost:3000/api/bills`)
            .then(response => {
                if (response.ok) {
                    response.json().then(json => dispatch(receive_latest_bills(json)));
                }
            });
    }
}
```

随后, 当我们调用 `store.dispatch(fetchLatestBills)` 后, 就会发起 action, 异步获取数据, 调用 reducer 函数返回更新 state, 新的 state
被 store 保存, 从而完成了对 store 的更新。如果这个过程能在服务器端就完成, 那么我们就能完成服务器端 store 的初始化了。

考虑到方便前后端调用想用的代码, 一种比较方便的方法就是把拉取数据的逻辑写在 React Class 的静态方法上。一方面服务端上可以通过直接操作静态方法
来提前拉取数据再根据数据生成 HTML，另一方面客户端可以在 componentDidMount 时去调用该静态方法拉取数据。

```javascript
// LatestBills/index.js

import React, {Component} from 'react';
import {connect} from 'react-redux';
import {Link} from 'react-router';

import {fetchLatestBills} from './actions';

class LatestBills extends Component {

    static fetchData({dispatch}) {
        return dispatch(fetchLatestBills());
    }

    componentDidMount() {
        const {dispatch} = this.props;
        this.constructor.fetchData({dispatch});
    }

    render() {

        // .....

    }

    function mapStatetoProps(state) {
        // ...
    }

    export default connect(mapStatetoProps)(LatestBills);
}
```

### React-Router

[react-router](https://github.com/reactjs/react-router) 通过一种声明式的方式匹配不同路由决定在页面上展示不同的组件。

这个简单的web应用可以这样定义路由:

```javascript
// routes.js

import React from 'react';
import {IndexRoute, Route} from 'react-router';

import Container from './component/Container';

import LatestBills from './containers/LatestBills';
import DetailedBill from './containers/DetailedBill';
import NotFound from './containers/404';

export default(
    <Route path="/" component={Container}>
        <IndexRoute component={LatestBills}/>
        <Route path="bill/:billId" component={DetailedBill}/>
        <Route path="*" component={NotFound}/>
    </Route>
)
```

## Server Side Render

就像前面已经介绍的那样, 服务器现在的工作就是在服务器端根据请求的 url 判断首屏需要哪些组件, 然后把这些组件需要用到的 state 提前在服务器端准备好
并发送给客户端。

我这里使用了`react-router`的 `match()` 方法, 将 request.url 匹配到我们之前定义的 routes。

```javascript
// server.js

const reactView = function *() {

    const [redirectLocation, renderProps] = yield thunkify(match)({routes, location: this.request.url});

    // 初始化store
    const store = applyMiddleware(thunkMiddleware)(createStore)(rootReducer, {});

    // 得到需要拉取数据的组件
    const components = renderProps.components.filter(
        component => (typeof component.fetchData === 'function')
    );

    // 拉取数据更新 store
    const promises = components.map(component =>
        component.fetchData({
            dispatch:store.dispatch, params: renderProps.params
        })
    )

    yield Promise.all(promises);

    yield new Promise((resolve, reject) =>
        setTimeout(() => {
            resolve();
        }, 0)
    )

    // 用 renderToString 将对应的组件树生成 HTML String
    const html = ReactDOMServer.renderToString(
        <Provider store={store}>
            <RouterContext {...renderProps}/>
        </Provider>
    );


    // 将 html string 和 store 的 state 返回给客户端
    yield this.render('index', {
        content: html,
        state: JSON.stringify(store.getState())
    });

};

```

我这里使用了 ejs 模板

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Koa React SSR</title>
    <link rel="stylesheet" href="/css/style.min.css">
</head>
<body>

<div id="app"><%- content %></div>
<script>window.__INITIAL_STATE__ = <%- state %></script>
<script src="/js/app.min.js"></script>
</body>
</html>
```

这样, 服务器端提前拉取了数据,并根据数据生成了html 直出返回给客户端, 用户访问时首屏内容就直接可见了。并且,我们把数据也跟着页面一起返回给了客户端,
客户端使用该数据去render, 从而与服务器上的数据状态保持一致。

```javascript
// client.js

import React from 'react';
import ReactDOM from 'react-dom';

import {Router, browserHistory} from 'react-router';
import {Provider} from 'react-redux';

import configureStore from './redux/configureStore';
import route from './routes';

// 通过服务器传来的数据初始化store, 从而和服务器数据状态保持一致
const initialState = window.__INITIAL_STATE__;
const store = configureStore(initialState);

ReactDOM.render(
    <Provider store={store}>
        <Router history={browserHistory}>
            {route}
        </Router>
    </Provider>
    , document.getElementById("app"));
```

注意到生成的 html 中有一个 `data-react-checksum` 的属性。

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-09-02%20at%208.51.51%20PM.png)

这个属性就是用来比较服务器端生成的 html 和客户端生成的一致。一致则不重新render，
省略创建DOM和挂载DOM的过程，接着触发 componentDidMount 等事件来处理服务端上的未尽事宜(事件绑定等)，从而加快了交互时间；
不一致时，则说明客户端 render 时的 state 和服务器上 render 时的 state 不一致, 这时组件将在客户端上被重新挂载 render, 交互时间变长, 这
种情况下浏览器的控制台中会出现警告。

![](http://ocd7f3wcw.bkt.clouddn.com/Screen%20Shot%202016-09-02%20at%209.05.10%20PM.png)

## 结语

到这里, React 同构直出的基本做法就完成了, 但是在实际的开发应用过程中还是可能会遇到很多其他的坑, 以后再总结一下来填坑吧。

[文中案例完整代码下载 >>](https://github.com/lancebiu/koa-react-redux-server-side-render)
