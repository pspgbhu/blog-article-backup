---
title: React 服务端渲染与同构
comments: true
date: 2018-03-09 01:45:58
categories: React
tags: Server Side Render
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

## 服务端渲染

### React API 的支持

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

此外 React v16 还提供了新的 render API 来专门为服务端渲染来做首屏渲染优化：
**ReactDOM.render 被 [ReactDOM.hydrate](https://reactjs.org/docs/react-dom.html#hydrate) 代替**（如果一个节点上有服务端渲染的标记，则 React 会保留现有 DOM，只去绑定事件处理程序，从而达到一个最佳的首屏渲染表现）。

### 实现服务端渲染并搭配 React Router

服务端渲染的原理就是：在服务端使用 React API 输出 HTML 文档，然后通过模板引擎将文档内容渲染在页面中，并输出到前端。

**React-Router 提供了专门的 StaticRouter 来进行服务端渲染。**

我使用了 Koa 来开发后台应用，在 Koa 的路由文件中将 React 输出的 HTML 返回到前端页面中：
```js
// server-side ./routes/index.jsx

import Router from 'koa-router';
import { renderToString } from 'react-dom/server';
import { StaticRouter } from 'react-router-dom'

const router = Router();

/**
 * 为了配合浏览器端的 React-Router 中 BrowserRouter 路由
 * 我们使用了 '*' 来匹配任何 URL，以返回同样的 HTML。
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

koa-router 通过 ejs 模板引擎将 React 输出的 HTML 内容渲染至页面。

```ejs
<!-- index.ejs -->

<!DOCTYPE html>
<html>
  <head>
    <title>React Isomorphic</title>
    <link rel='stylesheet' href='/css/style.css' />
  </head>
  <body>
    <div id="app"><%- html %></div>
    <script src="/js/app.js"></script>
  </body>
</html>
```
在客户端的入口文件，我们使用 hydrate 方法来渲染页面。hydrate 在服务端渲染的情况下，可以提供比 render 方法更好的首屏渲染性能。
```
// client-side index.jsx

import React from 'react';
import { hydrate } from 'react-dom';
import { BrowserRouter } from 'react-router-dom';
import App from '../common/App.jsx';

hydrate(
  <BrowserRouter>
    <App></App>
  </BrowserRouter>,
  document.querySelector('#app'),
);
```

这样就完成了一个最简单的服务端渲染。

### Redux 的服务端渲染

Redux 的服务端渲染实现思路也很清晰，就是在服务端初始化一个 Store 的同时，还需要把这个 Store 也传递到客户端。

#### 1. 在服务端构建初始 store

扩充 Koa 的路由文件：
```
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

#### 2. 模板引擎将初始的 store 渲染到页面中
模板引擎将 koa router 传来的 store 数据赋值给 `window.__INITIAL_STATE_` 对象下。

```ejs
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

#### 3. 客户端获取 Redux store 的初始值

```jsx
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

## 同构过程需要注意的地方

### 在服务端运行 JSX

服务端是不可以直接运行 JSX 的，但是我们可以借助于 babel-register 来在服务端支持 jsx 格式的文件。

在整个 node 程序的最上方引入 babel-register

```js
// app.js
require('babel-register')({
  presets: [
    'react',
  ],
  plugins: [
    ['transform-runtime', {
      polyfill: false,
      regenerator: true,
    }],
  ],
  extensions: ['.jsx'],
});

const Koa = require('koa')
const app = new Koa();
```
这样就能正常处理 .jsx 后缀的文件和 jsx 语法了。

### 样式文件的处理

通常我们在前端开发时，会直接在 jsx 中引入 css, less 等样式文件。
```
// app.jsx
import './style/app.less';
import './style/style.css';
```
但是，服务端却不能正确的处理这些文件，会引起服务端报错。

渲染 HTML 文档本身不需要 CSS 样式的参与，因此我们想办法忽略这些文件就可以了。

我们可以通过 babel-register 的插件 babel-plugin-transform-require-ignore 来忽略一些固定后缀的文件。
```
// app.js
require('babel-register')({
  presets: [
    'react',
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
  extensions: ['.jsx'],
});
```

### 分享实际项目的目录结构

以下是我 [React 同构模板](https://github.com/pspgbhu/react-isomorphic) 项目中的目录结构。

```bash
.
├── app.js                # 程序入口文件
├── bin
│   └── www               # 程序启动脚本
├── build                 # Webpack 配置
├── client                # Client Only Code
├── common                # 客户端和服务端共享代码, React 同构代码
│   ├── App.jsx           # React 入口文件
│   └── style             # 支持 less 样式
├── controllers           # Controllers
├── gulpfile.js           # Gulp 配置
├── middlewares           # Koa 中间件
├── public                # 静态资源文件
├── routes                # Koa 路由
├── utils                 # 工具函数
└── views                 # 页面模板文件

```

## 仍然需要改进的地方

以下是一些已知的需要处理的，但是尚未处理的场景，

### 1. 前端 JS 资源异步加载与 ejs 模板引擎动态渲染 script 标签。

如果项目工程比较大的情况下，所有的 JS 全部打包在一个文件中显然不是一个最佳的选择。

最佳的选择应该是按照页面或者模块打包成多个文件。首屏渲染时只加载当前页面所需的 JS 文件，跳转到其他页面时再异步加载其他页面对应的 JS 文件。

这里有一个痛点就是 ejs 中的 script 标签该怎么渲染。目前是写死的 script 标签：
```
<!-- index.ejs -->

<script src="/js/app.js"></script>
```
而最佳的实践应该是根据不同的页面路由返回不同的 script 标签。这里应该会涉及到 webpack 的打包和 Koa2 路由读取 webpack 配置中的 entry 属性的值。

以上这些问题我都会在之后慢慢补充正确的处理方案。
