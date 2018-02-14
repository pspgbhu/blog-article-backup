---
title: 前端优化：首屏渲染的阻塞与常见的优化方式
comments: true
date: 2018-02-01 00:12:12
categories: 前端优化
tags: 首屏渲染优化
img: http://static.zybuluo.com/pspgbhu/1euvd8zar7uxj5wmyh37ml9n/example.png
---

> 许多研究都表明，用户最满意的打开网页时间，是在2秒以下。用户能够忍受的最长等待时间的中位数，在6～8秒之间。这就是说，8秒是一个临界值，如果你的网站打开速度在8秒以上，那么很可能，大部分访问者最终都会离你而去。

影响页面打开的速度的因素有很多，比如：用户的网络质量、用户的浏览器、网页服务提供者服务器质量、服务器网络质量以及以及前端页面本身。

对于我们前端开发工程师而言，往往可控的就是前端页面本身，因此我们需要尽可能的缩短页面本身的渲染时间。

这篇文章将会介绍什么样的情况会影响到首屏渲染速度，并且会教给你一些常见的对应措施。

## 减少首屏白屏时间的关键

![IMG: rendering_engine.png](http://static.zybuluo.com/pspgbhu/x7fd0pvcydddz0v8vc37uuza/rendering-process.png)

根据布局树 (LayoutTree) 的构建流程我们可以知道，布局树 (LayoutTree) 的构建依赖于 DOM 树和 CSSOM 树。

因此任何导致 DOM 树或者 CSSOM 树构建被阻塞的行为，都最终将延长页面打开时白屏的时间，因此加速首屏渲染的关键就是避免 DOM 树和 CSSOM 树的构建被阻塞。

## CSS 对首屏渲染的影响

CSS 是用于描述页面 UI 的语言，页面的正常展示依赖于 CSS 规则的解析，下面就来说一下 CSS 对于页面首屏渲染的影响。

### 对布局树 (LayoutTree) 构建的阻塞

布局树的构建需要去遍历 DOM 树及其 DOM 节点对应的 CSSOM 节点，因此布局树的构建依赖于 CSSOM 的构建，CSSOM（CSS 对象模型）的构建又需要解析 CSS，所以**CSS 会阻塞布局树的构建，进而阻塞首屏渲染**。

CSS 解析的时间开销往往可以忽略不计，而通常真正产生影响的是：引入一个外部 CSS 文件时所花费在网络请求上的时间。因此 **减少 CSS 网络请求时间开销可以加快首屏渲染**。

值得一提的是：在 Chrome Blink 渲染引擎上即使 CSS 在文档中处于 HTML 元素的后面，还是会阻止其之前的元素显示。但是在 Safari 上没有这种现象。

现在许多人使用打包工具（如：Webpack 等）将一些图片转成 base64 放在 CSS 文件中以减少页面上发起的网络请求，但你一定要清楚这样的坏处：增加了 CSS 的体积，CSS 文件需要更长的请求时间，因此增加了首屏白屏的时间。

> CSS 阻塞渲染的特性是完全合乎情理的，浏览器为了避免在样式规则不完全的情况下显示出错误的文档内容，因此要求先解析 CSS 样式，然后才会开始渲染页面。

## JavaScript 对首屏渲染的影响

### 阻塞 DOM 构建

JavaScript 允许我们在 DOM 中创建、样式化、追加和移除新元素，即使该元素可能未出现在布局树中。但是我们在脚本中的同步代码却无法找到位于脚本之后的元素，这透露出一个重要的事实：**脚本在文档的何处插入，就在何处执行**。当 HTML 解析器遇到一个 script 标记时，它会暂停构建 DOM，将控制权移交给 JavaScript 引擎；等 JavaScript 引擎运行完毕，浏览器会从中断的地方恢复 DOM 构建。

或者，稍微换个说法：**执行脚本会阻止 DOM 构建，也就延缓了首次渲染。**

在网页中引入脚本的另一个微妙事实是，它们不仅可以读取和修改 DOM 属性，还可以读取和修改 CSSOM 属性。如果浏览器尚未完成 CSSOM 的下载和构建，而我们却想在此时运行脚本，会怎样？答案很简单，对性能不利：**浏览器将延迟脚本执行和 DOM 构建，直至其完成 CSSOM 的下载和构建。**

正是因为如此，所以我们常常将 script 标签放在 body 的结尾处。

### 阻塞用户首次交互

JavaScript 是单线程的，如果在初始化页面的时候，你的代码中存在大量同步运行的代码，导致 JS 线程一直处于繁忙状态，这时候用户在页面上进行交互时将不会得到任何反应，就像是卡死了一样。

如果说初始化时有大量的计算需要进行，请使用多个 `setTimeout` 进行分割，使之成为多个异步操作，这样在用户进行页面操作的时候，浏览器在多个异步操作间是可以有一个很短暂的时间来响应用户的 UI 操作的。

## 在 Chrome 上的一些特殊表现

写这一章节主要是因为在实际的测试中发现 Chrome 对首页渲染阻塞行为有着一些额外的处理情景。不过这一章节内容不必深究，只需要了解这种特性就好。

- 即使 CSS 在 HTML 元素后引入，依然是能够阻塞页面渲染的（有例外）。

- 这一条就是上面所说的例外：文档中引用 **外部** JS 文件，会使其 HTML 文档中之后的 CSS 样式丧失渲染阻塞特性，即使该 CSS 文件在某个 HTML 元素之前，依旧不能阻塞该元素的渲染。这也是上一条特性的例外情况。

对于第一种情况我能理解，毕竟希望 DOM 总是在 CSS 文件加载解析完成后才呈现。至于第二种表现，我还是不太理解浏览器这么设计的目的是什么。

## 图片资源的加载

### 1. 图片资源是在布局树构建时才开始解析下载

先看下面这个例子：在 CSS 文件后通过 background-image 样式属性来引入图片资源。

```html
<link rel="stylesheet" href="/static/style.css">
<div id="div-outer" style="display: none; background-image: url('/static/outer.png');"></div>
```

我们要观察它们的加载顺序并不难，通过 Chrome DevTools 里的 Network 便能一目了然：

![IMG: CSS & IMG](http://static.zybuluo.com/pspgbhu/cqfxu9u7oh4aw5z2qp8so6gs/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-02-05%20%E4%B8%8B%E5%8D%888.33.33.png)

很明显，在 CSS 文件加载完成后图片资源才开始引入的。但这是为什么呢？

请看下图：

![css-request-img.png-123.3kB](http://static.zybuluo.com/pspgbhu/jm5na1w7893nt9xcsn8m84i8/css-request-img.png)

CSS 中的图片资源是在 Recalculate Style 阶段开始下载的。即 **CSS 中的图片资源是在布局树构建时才开始解析下载的。**

在布局树构建阶段，浏览器会去遍历整个 DOM 树并计算相应节点的样式规则，在样式属性中所引入的图片资源也是在这个阶段才开始下载的，因此在 CSS 文件没有下载完成之前布局树是无法构建的，所以上面的图片才会在 CSS 文件下载完成后才开始加载。

### 2. 外层包裹 display: none 的元素，则该元素 CSS 中引入的图片不会被加载

现在我们来看另外一个例子：

```html
<div style="display: none;">
  <div id="div-inner" style="background-image: url('/static/inner.png');">
  </div>
</div>

<div id="div-outer" style="display: none; background-image: url('/static/outer.png');"></div>
```

让我们再看一下 Chrome DevTools

![IMG: CSS & IMG](http://static.zybuluo.com/pspgbhu/cqfxu9u7oh4aw5z2qp8so6gs/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-02-05%20%E4%B8%8B%E5%8D%888.33.33.png)

和上一个例子一模一样，只有 outer.png 被加载了。

仔细观察一下 `#div-inner` 是被包裹在一个 `display: none` 的 div 元素下。

当浏览器去遍历 DOM 树来构建布局树的时候，如果一个 DOM 元素的样式属性为 `display: none` 时，该元素是不会再布局树中生成布局对象的，因此浏览器只遍历了该元素的最外一层，对于其子元素浏览器不会再去遍历。

只有在读取到了该 DOM 元素对应的 CSS 属性有图片资源引用时，浏览器才会去下载。像上面这个例子，`#div-inner` 其父元素是 `display: none` 导致 `#div-inner` 元素不会在布局树构建阶段被遍历到，因此浏览器便不会去加载这个图片。

### 3. img 标签引入的图片会在预解析阶段处理

我们再看看这个例子：

```html
<div style="display: none;">
  <img src="/static/img_inner.png" alt="">
</div>
```

![屏幕快照 2018-02-05 下午8.41.12.png-23kB](http://static.zybuluo.com/pspgbhu/m96w9nwd3ds03j5ok0dmi55q/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-02-05%20%E4%B8%8B%E5%8D%888.41.12.png)

由 img 引用的图片在 HTML 解析阶段就会由预解析器解析，并请求会相应的数据。

By the way， 现在浏览器 img 标签 src 设置为空，并不会发起一次空的请求，但是为了兼容旧版浏览器，还是建议不要这么做。

## 一些首屏渲染的优化方案

- **link 标签尽量放在文档前面，script 标签尽量放在文档后面**
  这个不用多说，基本都已经成为了行业的规范。但是一定要了解这么做的原因是什么。

- **异步加载首屏非必要的 CSS 资源**
  CSS 的加载会阻塞页面布局树的构建，有些样式并不是首屏显示所必须的，我们通过专门的 JavaScript 代码来将其引入，这样该文件就不会阻塞布局树的构建了。

- **内联阻塞渲染的 CSS 样式**
  为获得最佳性能，可以考虑将关键 CSS 直接内联到 HTML 文档内。这样可以减少引用外部网络资源的时间。

- **避免使用 CSS import**
    一个样式表可以使用 CSS import (@import) 指令从另一样式表文件导入规则。不过，应避免使用这些指令，因为它们只有在收到并解析完带有 @import 规则的 CSS 样式表之后，才会发现导入的 CSS 资源。

- **避免将过多，过大的图片转为 base64 放在 CSS 文件中**
  将图片转为 base64 放在 CSS 文件中虽然可以减少网络请求次数，但与此同时也会增加 CSS 文件的体积，因此会造成更长的 CSS 文件加载时间。

- **Lazylod 首屏不需要的图片**
  避免图片加载占用过多的宽带资源，从而加快 CSS 文件的加载。

- **异步加载首屏非必须的 JS 文件**
  用一段专门的 JS 来异步插入 script 标签。首先可以避免 JS 对 DOM 构建的阻塞，同时还可以节约的宽带资源，从而加快 CSS 文件的下载。

- **异步执行首屏非必须的 JS 代码**
  可以使用 `defer`, `async` 属性来延迟执行 JS 代码的时机。

- **避免运行时间长的 JavaScript 同步代码**
   运行时间长的 JavaScript 会阻止浏览器构建 DOM、CSSOM 以及渲染网页，所以任何对首次渲染无关紧要的初始化逻辑和功能都应延后执行。如果需要运行较长的初始化序列，请考虑将其拆分为若干阶段，以便浏览器可以间隔处理其他事件。

> 在实践中并不需要死守上面的教条，比如说：“使用内联样式” 虽然会减少白屏时间，但却不利于项目的开发维护。因此在实际开发中合理的取舍就显得尤为重要。

## 写在最后

本文也只是片面的分析了一下常见的渲染阻塞问题，与相应的解决方式，如果发现本文还有其他没有提及到的首屏优化方案，欢迎留言补充。

  [1]: http://static.zybuluo.com/pspgbhu/9qjyb7f6dmyuf1mdhfigpakv/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-01-31%20%E4%B8%8A%E5%8D%881.04.01.png
  [2]: http://static.zybuluo.com/pspgbhu/1euvd8zar7uxj5wmyh37ml9n/example.png
