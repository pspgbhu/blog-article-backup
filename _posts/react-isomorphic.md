---
title: React 服务端渲染与同构
comments: true
date: 2018-03-09 01:45:58
categories: React
tags: Server-Side-Render
img: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1520541763181&di=5421f1ab7a5b72340d02b021493d244b&imgtype=0&src=http%3A%2F%2Fs1.51cto.com%2Fwyfs02%2FM01%2F88%2F7F%2FwKiom1f55HCSS-DrAACSkyHme8o914.png-wh_651x-s_1436211364.png

---

近日实现了一个 React 同构直出的模板 [React Isomophic](https://github.com/pspgbhu/react-isomorphic)，开箱即用。

该模板支持 Koa2 + React + React Router + Redux + Less 。

## 为什么要服务端渲染和同构

传统的 SPA 开发模式由于其页面渲染全部放在了客户端，从而导致了一些一直以来难以解决的痛点：首屏白屏时间较长、SEO 不友好等。

而服务端渲染则可以解决这些传统的痛点：

1. 服务端直出 HTML 文档，让搜索引擎更容易读取页面内容，有利于 SEO。
2. 不需要客户端执行 JS 就能直接渲染出页面，大大减少了页面白屏的时间。

服务端渲染的好处上面已经描述清楚了，那么同构呢？

感谢 Nodejs 的出现让服务端也有了运行 JavaScript 的能力，这样我们原本在前端运行的 React 在后端也可以运行了。两端公用一套逻辑代码，减少了开发量，也避免了前后端页面渲染的不一致。

## React API 的支持

React 提供了四个 API 来将虚拟 DOM 输出为 HTML 文本。

前两个方法在浏览器和服务端都是可用的：

- [ReactDOMServer.renderToString(element)](https://reactjs.org/docs/react-dom-server.html#rendertostring)
- [ReactDOMServer.renderToStaticMarkup(element)](https://reactjs.org/docs/react-dom-server.html#rendertostring)

下面这两个方法会将文本以流的形式输出，因此只能在服务端运行。

- [ReactDOMServer.renderToNodeStream(element)](https://reactjs.org/docs/react-dom-server.html#rendertostring)
- [ReactDOMServer.renderToStaticNodeStream(element)](https://reactjs.org/docs/react-dom-server.html#rendertostring)

**renderToString 方法与 renderToStaticMarkup 的区别：**

后者不会在创建出的 DOM 节点上添加任何的 React 属性，这也就意味着创建出的页面将不会具备响应式的特性，`renderToStaticMarkup` 方法适用于创建纯静态页。

我这边需要在客户端继续使用 React 的响应式特性，因此我选用了 `renderToString` 方法。

## 让服务端支持 JSX

Node 环境下是不可以直接运行 JSX 的，但是我们可以借助于 babel-register 来在服务端支持 jsx 格式的文件。

在整个 node 程序的最最开始引入 babel-register:

```js
// app.js
require('babel-register')({
  presets: [
    'es2015',
    'react',
    'stage-2',
  ],
  plugins: [
    ['transform-runtime', {
      polyfill: false,
      regenerator: true,
    }],
  ],
  extensions: ['.jsx', '.js'],
});

const Koa = require('koa')
const app = new Koa();

//...
//...
```

这样就能正常处理 .jsx 后缀的文件和 jsx 语法了。同时还支持了 import 等 ES6 语法。

## 项目框架搭建

实现服务端渲染的第一步先从搭建起一个后端项目开始。为了减少 Node 项目的搭建成本，这里我推荐使用 [koa-generator](https://github.com/17koa/koa-generator)。

在脚手架的基础上，再新建三个文件夹：

- `/build`：
  Webpack 启动配置，主要用于打包浏览器端所需要的静态资源。
- `/common`：
  主要是 React 的组件和逻辑代码。前后端公用这一部分代码。通常我们将 React 的根组件作为整个 common 文件夹的入口暴露出去。
- `/client`：
  只在浏览器端运行的文件。

```bash
├── app.js                # 程序入口文件
├── bin
│   └── www               # 程序启动脚本
├── build                 # Webpack 配置，用于打包前端静态资源
├── client                # 只在浏览器端运行的代码
|   └── index.jsx         # 浏览器端的入口文件
├── common                # 客户端和服务端共享代码, React 同构代码
|   └── App.jsx           # React 根组件
├── controllers           # Controllers
├── public                # 静态资源文件
├── routes                # Koa 路由
└── views                 # 页面模板文件
    └── index.jsx         # ejs 渲染模板
```

## 实现服务端渲染

单单实现 React 服务端渲染非常简单：使用 React 的 `renderToString` API 将虚拟 DOM 输出为 HTML 字符串，然后当浏览器请求页面时返回给浏览器就 OK 了。

```js
// Node 服务端  ./routes/index.jsx

import Router from 'koa-router';
import { renderToString } from 'react-dom/server';
import App from '../common/App';
const router = Router();

/**
 * 为了配合浏览器端的 React-Router 中 BrowserRouter 路由
 * 我们使用了 '*' 来匹配任何 URL，以返回同样的 HTML。
 */
router.get('/', async (ctx) => {
  // 因为使用了 babel-register，因此我们可以直接使用 jsx 语法。
  // 将根组件 App 输出为 html 字符串。
  const content = renderToString(<App/>);

  await ctx.render('index', {
    html: content,  // 将 html 字符串通过 ejs 模板渲染出来
  });
});

export default router;
```

对应的 index.ejs 文件：

```html
<!-- index.ejs -->

<!DOCTYPE html>
<html>
  <head>
    <title>React Isomorphic</title>
    <link rel='stylesheet' href='/css/style.css' />
  </head>
  <body>
    <!-- koa-router 将 react 虚拟dom 输出为字符串，渲染在 html 变量中 -->
    <div id="app"><%- html %></div>
    <script src="/js/app.js"></script>
  </body>
</html>
```
这样，在浏览器请求页面的时候，服务端就会返回具备完整 DOM 结构的 HTML 了。

服务端渲染只负责渲染出首屏的内容，往往在首屏渲染后页面上还是会有很多的交互和异步操作的，因此我们依然需要向之前前端开发一样，打包 JS 并在 HTML 中引入对应的 JS 文件。

`/client/index.jsx` 作为整个浏览器端的入口 JS 文件，内容大致如下：

```js
// ./client/index.ejs

import React, { Component } from 'react';
import { hydrate } from 'react-dom';
import { BrowserRouter, withRouter } from 'react-router-dom';

// 用 hydrate 来替代 render
hydrate(
  <App></App>,
  document.querySelector('#app'),
);
```
> React v16 提供了新的 render API 来专门为服务端渲染来做首屏渲染优化：
> [ReactDOM.hydrate](https://reactjs.org/docs/react-dom.html#hydrate) (如果一个节点上有服务端渲染的标记，则 React 会保留现有 DOM，只去绑定事件处理程序，从而达到一个最佳的首屏渲染表现）。
> 我们将使用 hydrate 来替代 render

至此，一个最简单的服务端渲染就完全构建完毕了。

## 搭配 React Router 使用

浏览器端的 React Router 支持两种模式:

- Hash Router：基于 URL Hash 来实现的 Router。
- Browser Router：看起来更像是真实的链接跳转，但需要服务端的支持。

这里我选择了 Browser Router，它会让我在使用 React Router 跳转时有一种更真实的跳转的感觉。

这里我们需要对浏览器端和服务器端分别处理，对于浏览器端：

```js
// ./client/index.ejs

import React, { Component } from 'react';
import { hydrate } from 'react-dom';
import { BrowserRouter, withRouter } from 'react-router-dom';

// 用 hydrate 来替代 render
hydrate(
  <BrowserRouter>
    <App></App>
  </BrowserRouter>,

  document.querySelector('#app'),
);
```

**React-Router 提供了专门的 StaticRouter 来进行服务端渲染。** 我们将浏览器请求的页面路径作为参数传入到 StaticRouter 的 location 参数中，StaticRouter 就会帮助我们来返回不同的虚拟 dom 节点。然后我们再用 `renderToString` 方法来转成 html 字符串。

同时我们还需要把 `router.get('/')` 更改为 `router.get('*')`。任何路径的请求都将会进入到该路由进行处理。

```js
// Node /routes/index.jsx

import Router from 'koa-router';
import { renderToString } from 'react-dom/server';
import { StaticRouter } from 'react-router-dom'

const router = Router();

/**
 * 为了配合浏览器端的 React-Router 中 BrowserRouter 路由
 * 我们使用了 '*' 来匹配任何来自浏览器的请求，这样保证了任何路径的请求都将得到该路由的处理。
 */
router.get('*', async (ctx) => {
  const context = {};
  const content = (
    <StaticRouter location={ctx.url} context={context}>
      <App/>
    </StaticRouter>
  );

  await ctx.render('index', {
    html: content,
  });
});

export default router;
```

### Redux 的服务端渲染

Redux 的服务端渲染实现思路也很清晰，就是在服务端初始化一个 Store 的同时把这个 Store 也传递到客户端。

#### 第一步：在服务端构建初始 store

扩充 Koa 的路由文件：

```js
// server-side ./routes/index.jsx

import Router from 'koa-router';
import { renderToString } from 'react-dom/server';
import { StaticRouter } from 'react-router-dom';
import { Provider } from 'react-redux';
import { createStore } from 'redux'

const router = Router();

router.get('*', async (ctx) => {
  // 在服务端初始化 store 的数据
  const store = createStore(state => state, {
    name: 'Pspgbhu',
    site: 'http://pspgbhu.me',
  });

  const context = {};
  const content = (
    <StaticRouter location={ctx.url} context={context}>
      {/* 增加 Provider */}
      <Provider store={store}>
        <App/>
      </Provider>
    </StaticRouter>
  );

  // 获取 store 数据对象
  const preloadedState = store.getState();

  await ctx.render('index', {
    html: content,
    state: preloadedState, // 将 store 数据传递给 ejs 模板引擎
  });
});

export default router;
```

#### 第二步：模板引擎将初始的 store 渲染到页面中

模板引擎将 koa router 传来的 store 数据赋值给 `window.__INITIAL_STATE_` 对象下。

```html
<!-- index.ejs -->

<!DOCTYPE html>
<html>
  <head>
    <title>React Isomorphic</title>
    <link rel='stylesheet' href='/css/style.css' />
  </head>
  <body>
    <div id="app"><%- html %></div>
    <script>
      // 将服务端的 store 对象赋值给该变量
      window.__INITIAL_STATE_ = <%- state %>;
    </script>
    <script src="/js/app.js"></script>
  </body>
</html>
```

#### 第三步：客户端获取 Redux store 的初始值

```js
// client-side index.jsx

import React from 'react';
import { hydrate } from 'react-dom';
import { BrowserRouter } from 'react-router-dom';
import { createStore } from 'redux';
import { Provider } from 'react-redux';

import App from '../common/App';

// 通过服务端注入的全局变量得到初始的 state
const preloadedState = window.__INITIAL_STATE_;

const store = createStore(state => state, preloadedState);

hydrate(
  <Provider store={store}>
    <BrowserRouter>
      <App></App>
    </BrowserRouter>
  </Provider>,

  document.querySelector('#app'),
);

```

## 样式文件的处理

通常我们在前端开发时，会直接在 jsx 中引入 css, less 等样式文件。

```js
// app.jsx
import './style/app.less';
import './style/style.css';
```

但是服务端却不能正确的处理这些文件，会引起服务端报错。 渲染 HTML 文档本身不需要 CSS 样式的参与，因此我们想办法忽略这些文件就可以了。

我们可以通过 babel-register 的插件 babel-plugin-transform-require-ignore 来忽略一些固定后缀的文件。下面让我们来扩充 app.js 中的 bable-register 配置。

```js
// app.js

require('babel-register')({
  presets: [
    'es2015',
    'react',
    'stage-2',
  ],
  plugins: [
    ['transform-runtime', {
      polyfill: false,
      regenerator: true,
    }],
    // babel-plugin-transform-require-ignore 插件
    // 可以使 node 忽略一些固定的后缀文件。
    [
      'babel-plugin-transform-require-ignore', {
        extensions: ['.less', '.sass', '.css'],
      },
    ],
  ],
  extensions: ['.jsx', '.js'],
});
```

## 开发环境和生产环境下的静态资源打包

上面的内容都是在说服务端渲染的事情，当页面在浏览器端运行的时候还是需要在浏览器中加载 JS 静态资源的。这些静态资源我们通常使用 Webpack 进行打包。

### 生产环境下的静态资源

生产环境下处理静态资源的打包很简单，直接使用 webpack 打包出对应的 JS 和 CSS，然后在 ejs 模板中直接引用对应的静态资源即可。

### 开发环境下的静态资源

开发环境下，因为需要频繁的更改项目代码，所以很需要代码的热更新。 服务端 Node 代码的热更新通过 nodemon 就能轻松实现。

对于前端静态资源的热更新，我是通过 webpack 去 watch 相应的代码源码，每次检测到代码改动后 webpack 再自动去 rebuild 代码。

此外，在开发环境下，我会将静态资源全部都打包在 `/.dev` 目录下，而不是 `/build` 目录下。并且在 app.js 中进行如下设置

```js
/**
 * 非生产环境下引用 .dev 目录作为静态资源目录。
 * .dev 目录下的资源会优先于 public 文件下的资源。
 */
if (process.env.NODE_ENV !== 'production') {
  app.use(serve(path.join(__dirname, '.dev')));
}

app.use(serve(path.join(__dirname, 'public')));
```
以此来区分不同环境下的静态资源引用。

## 一些要注意的地方

### 1. 生命周期的不同

在同构的基础上，Node 服务端和浏览器端虽然都公用了整个 `/common` 目录下的 React 组件，但是在具体运行中还是有些不同。

在 Node 服务端，组件的生命周期只走到了 `componentWillMount()`。

在浏览器端，组件拥有着完整的生命周期。

### 2. Node 环境下没有 window 对象等其他一些浏览器端特有的全局对象或方法。

平时我们在做前端开发的时候或多或少都会用到 window 对象和其他一些浏览器端特有的全局对象（如 document 等）， Node 环境下是没有这两个对象的，如果贸然的使用，肯定是会引起 Node 端的报错的。

那么在组件中是不是不能用这些对象了呢？当然不是。上面有提到 Node 环境下，组件的生命周期只能走到 `componentWillMount()`，再之后的生命周期函数就不会被调用了。因此我们就可以把这些浏览器端特有的对象和方法放在 `componentWillMount()` 之后的生命周期函数里使用，比如在 `componentDidMount()` 中使用 `document.querySelector('#app')` 方法。

## 写在最后

整的来说，由于 React 及其全家桶对服务端渲染的支持十分不错，因此在服务端渲染的实现上整体还是比较简单的。而且实现上也很灵活，肯定不止本文的这一种思路。

最近还重构了我的[博客](http://blog.pspgbhu.me)，就是基于 React 服务端渲染的，体验感觉还是不错的，各位看官可以体验一下。