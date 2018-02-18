---
title: 浏览器渲染引擎工作原理（二）：布局与绘制
comments: true
date: 2018-02-18 23:06:22
categories: 浏览器
tags: 浏览器渲染引擎
img: http://static.zybuluo.com/pspgbhu/i5oob324pu3mzb57ptobkn7m/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-01-20%20%E4%B8%8B%E5%8D%887.30.18.png
---

![rendering_engine.png][1]

## 布局阶段

> 当布局树构建完毕后，布局树上并没有每个对象在设备视口内的确切位置和大小，而为它确定这些信息的过程我们称之为 **布局（Layout）**。

![屏幕快照 2018-01-20 下午7.30.18.png-21.6kB][6]

**盒模型** 就是在 **布局阶段** 计算并产生的。浏览器会遍历整个 **布局树**，来为每一个 **布局对象** 计算出对应的 **盒模型数据**。

仅仅有 **盒模型** 还是不能够准确的将每一个 **布局对象** 准确的绘制到浏览器视口内，浏览器会继续去计算每一个元素在页面中的坐标值，即 x 与 y 值。

Layout 涉及大量的页面计算，往往页面性能瓶颈也是出现这里，因此关于优化这里大有文章可做。这篇文章也会就重排与重绘进行一些讨论。

## 绘制阶段

### 渲染层 (PaintLayer) 及渲染层合并 (Composite)

> 绘制 (Paint) 是填充像素的过程。它涉及绘出文本、颜色、图像、边框和阴影，基本上包括元素的每个可视部分。绘制一般是在多个渲染层上完成的。

“绘制一般是在多个渲染层上完成的”，这里所说的 **渲染层** 在 WebKit 中被称作 **RenderLayer** 在 Blink 中称之为 **PaintLayer**。而将多个渲染层进行合并的步骤称之为 **渲染层合并 (Composite)**。

> 渲染层合并 (Composite)： 由于页面的各部分可能被绘制到多层，由此它们需要按正确顺序绘制到屏幕上，以便正确渲染页面。对于与另一元素重叠的元素来说，这点特别重要，因为一个错误可能使一个元素错误地出现在另一个元素的上层。

一般来说，具有相同坐标系的 **布局对象 (LayoutObject)**，隶属同一个 **渲染层 (PaintLayer)**。渲染层的存在使得页面内的元素能够按照正确的顺序合成，以便能够正确的显示重叠的内容，比如半透明的元素等。有一些特定的条件可以为一些特殊的布局对象创建一个新的渲染层。

根据创建 PaintLayer 的原因不同，可以将其分为常见的 3 类：

- **NormalPaintLayer**
    - 根元素（HTML）
    - 有明确的定位属性（relative、fixed、sticky、absolute）
    - 透明的（opacity 小于 1）
    - 有 CSS 滤镜（fliter）
    - 有 CSS mask 属性
    - 有 CSS mix-blend-mode 属性（不为 normal）
    - 有 CSS transform 属性（不为 none）
    - backface-visibility 属性为 hidden
    - 有 CSS reflection 属性
    - 有 CSS column-count 属性（不为 auto）或者 有 CSS column-width 属性（不为 auto）
    - 当前有对于 opacity、transform、fliter、backdrop-filter 应用动画
    - `<video>` 元素
    - 具有3D（WebGL）上下文或者硬件加速的2D上下文的 `<canvas>` 元素


- **OverflowClipPaintLayer**
    - overflow 不为 visible

- **NoPaintLayer**
    - 不需要 paint 的 PaintLayer，比如一个没有视觉属性（背景、颜色、阴影等）的空 div。

满足以上条件的 **布局对象** 会拥有独立的渲染层，而其他的 **布局对象** 则和其第一个拥有 **渲染层** 的父元素共用一个。


### 合成层 (GraphicsLayer)

如果为每一个渲染层都单独的绘制一遍 “背衬”(backing surface)，这将会浪费大量的内存，因此浏览器会将一些（并不是全部）渲染层组成一个共同的 **合成层 (GraphicsLayer)**。每个合成层都有相关的渲染层所绘制的 **图形上下文 (GraphicsContext)**。合成器 (Compositor) 会将图形上下文的以位图的形式输出，最终由 GPU 将多个位图进行合成，最终将页面呈现在浏览器视口中。

和布局对象与渲染层的关系一样，每一个渲染层要么拥有自己的合成层，要么与其他渲染层一起共用一个来自于它们共同祖先的合成层。这里也有一些特定的条件，可以为特定的渲染层创建一个新的合成层，下面列举了一些（具体可以查看 Blink 源码 [CompositingReasons.cpp](https://chromium.googlesource.com/chromium/blink/+/master/Source/platform/graphics/CompositingReasons.cpp) ）：

- 进行3D或者透视变换的CSS属性
- 使用硬件加速视频解码的 `<video>` 元素
- 具有3D（WebGL）上下文或者硬件加速的2D上下文的 `<canvas>` 元素
- 组合型插件（即Flash）
- 具有有CSS透明度动画或者使用动画式Webkit变换的元素
- 具有硬件加速的CSS滤镜的元素
- 子元素中存在具有组合层的元素的元素（换句话说，就是存在具有自己的层的子元素的元素）
- 同级元素中有Z索引比其小的元素，而且该Z索引比较小的元素具有组合层（换句话说就是在组合层之上进行渲染的元素）

### 重排与重绘

除了在页面初始化时浏览器会绘制页面，同时我们还可以使用 JavaScript 来操作页面上的 DOM 元素及其 CSS 样式来完成各种炫酷的变幻效果。当浏览器执行完 JavaScript，并且再次去绘制页面时，开销是巨大的，并且很可能造成页面卡顿，影响体验。

下图称之为 **像素管道 (The pixel pipeline)**，管道的每个部分都有机会产生卡顿，因此务必准确了解您的代码触发管道的哪些部分。


![frame-full.jpg-12.3kB][7]

#### 1. JS / CSS > 样式 > 重排 > 重绘 > 合成

![frame-full.jpg-12.3kB][8]

如果修改了元素的“layout”属性，也就是改变了元素的几何属性（例如宽度、高度、左侧或顶部位置等），那么浏览器将必须检查所有其他元素，然后“自动重排”页面。任何受影响的部分都需要重新绘制，而且最终绘制的元素需进行合成。

#### 2. JS / CSS > 样式 > 重绘 > 合成

![frame-no-layout.jpg-10.9kB][9]

如果修改“paint only”属性（例如背景图片、文字颜色或阴影等），即不会影响页面布局的属性，则浏览器会跳过布局，但仍将执行绘制。

#### 3. JS / CSS > 样式 > 合成

![frame-no-layout-paint.jpg-9.7kB][10]

如果更改一个既不要布局也不要绘制的属性，则浏览器将跳到只执行合成。

仅仅触发合成步骤的开销最小，最适合于应用生命周期中的高压力点，例如动画或滚动。

> 注：如果想知道更改任何指定 CSS 属性将触发上述三个版本中的哪一个，请查看 [CSS 触发器](https://csstriggers.com/)。

### Compositor-Only 与流畅动画的最佳实践

#### 1. Compositor-Only（流畅动画的最重要条件）

正如前文所提到的，如果更改一个既不要布局也不要绘制的属性，则浏览器将跳到只执行合成。目前只有两个 CSS 属性满足 **“仅合成器” (Compositor-Only)**：

- **transform**
- **opacity**

因此坚持使用上面两个属性来完成各种动画，是一个简单且立竿见影的优化方式。

#### 2. 将动画元素提升到自己的层
此外，将即将进行动画的元素提升到自己的层，可以减少浏览器绘制区域。

创建新层的最佳方式是使用 will-change CSS 属性。并且通过 transform 的值将创建一个新的合成器层：

```css
.moving-element {
    will-change: transform;
}
```

此外，对于不支持 will-change 但受益于层创建的浏览器，如 IOS Safari < 9.3 的浏览器，可以使用 3D 变形来强制创建一个新合成层。

```css
.moving-element {
  transform: translateZ(0);
}
```

不过需要注意的是：虽然提升合成层有利于性能，**但是切勿滥用**，创建的每一层都需要内存和管理，因此只提升必要的合成层。

> 如无必要，请勿提升元素。


#### 3. 减少 JavaScript 的介入（这也是从杜绝卡顿的重要影响之一）

JavaScript 的执行处于 **像素管道** 的第一环，减少非必要的 JavaScript 的执行也是优化的重要手段。

![frame-full.jpg-12.3kB][11]

通常最好的动画方式是使用 CSS Animation。下面是一个简单的示例：

```css
@keyframes slidein {
  from {
    transform: scale(1);
  }

  to {
    transform: scale(0);
  }
}

.animation {
    animation: 3s linear slidein;
}

.target {
    width: 100px;
    height: 100px;
    will-change: transform;
}
```
```js
// 为 .target 元素来添加动画
document.querySelector('.target').classList.add('animation');
```

或者，用 `transform` + `transition` 来完成动画也是不错的：

```css
.target {
    width: 100px;
    height: 100px;
    transform: scale(1);
    transition: transform 1s linear;
    will-change: transform;
}

.scale {
    transform: scale(0);
}
```
```js
// 为 .target 元素来添加动画
document.querySelector('.target').classList.add('scale');
```

总而言之，就是**将动画的计算交给浏览器去做**，而不是在 JS 中做。

下面举一个极端的反面示例：
```css
.target {
    width: 100px;
    height: 100px;
}
```
```js
var el = document.querySelector('.target');
var w = 100;
var step = w / 60;

function scale() {
    w = w <= step ? 0 : w - step;
    el.style.width = w + 'px';
    el.style.height = w + 'px';
    if (w > 0) {
        window.setTimeout(scale, 1000 / 60);
    }
}

window.setTimeout(scale, 1000 / 60);
```
上面的这种写法依旧可以实现前面示例中的效果，但是却犯了很多个致命的错误，很有可能在执行动画时造成卡顿：

1. 首先是 JS 介入到了动画的每一帧的绘制，这就意味着每一帧动画的绘制都将要经过 “像素管道” 的流程。

2. 其次是使用了 width 和 height 属性来实现动画，这两个属性的改变将会触发元素的 Layout, Paint 和 Composite。因此每一帧的绘制都会触发页面的重排重绘，这往往也是流畅动画的性能瓶颈所在。

3. 最后需要一提的是，如果在某些特殊情况下需要使用 JS 来实现动画，请总是使用 `requestAnimationFrame` 来代替 `setTimeout` 和 `setInterval`。

## 写在最后
浏览器是网页寄宿的重要环境，了解浏览器的工作原理是每一个前端工程师必备的硬性素质之一。无论是页面渲染优化，或疑难杂症的排查都极度依赖于对浏览器工作原理的熟悉程度。本文基本上将浏览器渲染工作原理梳理了一遍，着重描述了布局和绘制阶段，并写下了自己在日常工作中所总结的优化技巧与最佳实践，希望能给大家带来一些帮助。

  [1]: http://static.zybuluo.com/pspgbhu/jiazxhpd7texfnxg6dh1zy2q/browser-rending.png
  [2]: http://static.zybuluo.com/pspgbhu/9mi6j7osg01apgzjc0298giv/parsing-model-overview%20%281%29.png
  [3]: http://static.zybuluo.com/pspgbhu/15hbugs0m5ret8oebay7s7dg/cssom-construction.png
  [4]: http://static.zybuluo.com/pspgbhu/mpktzkj3x8bgkjc3shgbovro/cssom-tree.png
  [5]: http://static.zybuluo.com/pspgbhu/94y07sfhf3qf8jsbzx1hco80/34839297-1d67a892-f73c-11e7-9477-97fc10b15745.png
  [6]: http://static.zybuluo.com/pspgbhu/i5oob324pu3mzb57ptobkn7m/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-01-20%20%E4%B8%8B%E5%8D%887.30.18.png
  [7]: http://static.zybuluo.com/pspgbhu/s66hxejrr60bggj0az2fkzfy/frame-full.jpg
  [8]: http://static.zybuluo.com/pspgbhu/s66hxejrr60bggj0az2fkzfy/frame-full.jpg
  [9]: http://static.zybuluo.com/pspgbhu/u9q94e2djmlinbbunayjbfiw/frame-no-layout.jpg
  [10]: http://static.zybuluo.com/pspgbhu/u20r4qqthi76qxdu5nc1nlq0/frame-no-layout-paint.jpg
  [11]: http://static.zybuluo.com/pspgbhu/s66hxejrr60bggj0az2fkzfy/frame-full.jpg
  [12]: http://static.zybuluo.com/pspgbhu/x7fd0pvcydddz0v8vc37uuza/rendering-process.png
  [13]: http://static.zybuluo.com/pspgbhu/e9suhdg3jtwdqaowsvx76b1a/browser-rendering.png
