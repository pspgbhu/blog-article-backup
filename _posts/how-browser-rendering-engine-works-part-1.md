---
title: 浏览器渲染引擎工作原理（一）：HTML 解析与树的构建
comments: true
date: 2018-01-22 01:22:58
categories: 浏览器
tags: 浏览器渲染引擎
img: http://static.zybuluo.com/pspgbhu/x7fd0pvcydddz0v8vc37uuza/rendering-process.png

---

“渲染引擎（Rendering Engine）”也称“排版引擎（Layout Engine）”，负责在浏览器的屏幕上显示请求的内容，是浏览器重要组成之一。本文将基于 Blink 渲染引擎来深入解析浏览器渲染工作的流程及原理。


## Blink 与 WebKit  的渊源

WebKit 是苹果公司主导研发的一款浏览器渲染引擎，并在 2005 年 6 月开源了该引擎。在 2008 年 Google 推出了 Chrome 浏览器，并且采用了 WebKit 作为其浏览器的渲染引擎。


根据提交统计，Google 自 2009 年年底以来一直是 WebKit 代码库的最大贡献者，Google 工程师们在实现标准上一直表现出了比较激进的态度，而 Apple 则一向比较保守，虽然 Google 的开发工程师提交大部分的 WebKit 更改，但是 WebKit 的最终决策权还是 Apple 的。同时像 Google 这种技术实力雄厚的公司也不愿意一直受制于人，希望在浏览器的开发上拥有更大的自由度，因此久而久之 Google 与 Apple 分道扬镳就是必然的事情了。

最终，Google 在 2013 年 4 月 宣布从 WebKit 分支出自己的浏览器渲染引擎 Blink。


## 渲染引擎的主要工作流程

下图展示了渲染引擎的主要工作流程。

![rendering_engine.png][1]


其主要分为下面四个步骤：

1. 解析 HTML 标记形成 DOM 树，解析 CSS 标记 形成 CSSOM 树。

2. DOM 树和 CSSOM 树将共同创建另一个树：布局树

3. 对布局树排版布局，来确定每个节点在视口内确切位置和大小。

4. 渲染引擎遍历整个布局树来将每个节点绘制出来。


## 解析和对象模型的构建

### HTML 解析 与 DOM 构建

浏览器渲染流程的开始通常是从一个 HTML 文档的请求开始的，下图来自 W3C 描述了 HTML 解析的大致流程：

![parsing-model-overview (1).png-10.1kB][2]

> **HTML 解析基本流程：网络 -> 字节 -> 字符 -> 标记化 -> 树构建**


1. 在 **Network** 阶段我们会根据浏览器 URL 来请求回相应的 HTML 文档；

2. **字节流解码器 (Byte Stream Decoder)** 按照请求回的 HTML 文件的字符编码格式将字节解码为字符；

3. 随后被解码的字符和来自直接操纵输入流的各种 API（如 `document.write()`）传入的字符一起放入了 **输入流预处理器 (Input Stream Preprocessor)**，以供后续标记化阶段使用；

4. 进入 **标记化阶段** 之后，**标记生成器** 会根据 **标记化算法** 来解析 HTML 文档，输出结果是 HTML 标记。**标记化算法** 根据 HTML 代码中 HTML 标签的闭合来创建 HTML 标记。值得一提的是，浏览器在解析 HTML 增添了许多容错机制，使浏览器能够解析不规范的 HTML 语法。

5. 在解析器创建的时候，也会创建 Document 对象。在 树构建阶段 **以 Document 为根节点的 DOM 树也会不断地进行修改**，向其中添加各种元素。标记生成器发送的每个节点都会由树构建器进行处理。当解析器解析到 script 标签时，主解析器会暂停之后的解析，直到脚本执行完毕。**如果在脚本中更改了 DOM 结构，解析器会重新解析文档**，因此 HTML 文档的解析往往是一个反复的过程。

如果希望进一步了解 HTML 文档解析，可以查看 [HTML5 规范 - HTML解析](https://www.w3.org/TR/html5/syntax.html)。


### HTML 解析的阻塞

- **script 标记及 JS 脚本的执行会阻塞 DOM 解析**。
    当 HTML 解析器遇到一个 script 标记时，它会暂停构建 DOM，将控制权移交给 JavaScript 引擎；等 JavaScript 引擎运行完毕，浏览器会从中断的地方恢复 DOM 构建。

- **如果 JavaScript 更改了 DOM 结构，则解析器将会重新解析部分 HTML 文档**。

### CSS 对象模型 (CSSOM)

**CSSOM** 即 **CSS 对象模型**，它定义了媒体查询，选择器和 CSS（包括通用解析和序列化规则）本身的 API。

与处理 HTML 时一样，我们需要将收到的 CSS 规则转换成某种浏览器能够理解和处理的东西。因此，我们会重复类似于构建 DOM 的步骤，不过这次是为了 CSS：

![cssom-construction.png-4.1kB][3]

CSS 规则的来源：

- 由 link 标签所请求回来的外部样式。
- 由 style 标签所定义的内部样式
- 直接定义在元素标签上的内联样式。
- 也有可能是由 JS 脚本所创建的 `CSSStyleSheet` 对象。

CSSOM 与 DOM 是分别独立的数据结构。与 DOM 树相同，**CSSOM 也构成了树状的结构**。CSSOM 为何具有树状结构？因为页面上的任何对象计算最后一组样式时，浏览器都会先从适用于该节点的最通用规则开始，然后通过应用更具体的规则以递归方式优化计算的样式。示意图如下：

![cssom-tree.png-14.2kB][4]

同时，CSS 规则与 DOM 节点的匹配是一个相当复杂和有性能问题的事情。所以，我们常常能再各个地方看到一些忠告：DOM 树要小，CSS 尽量使用 id 和 class，千万不要过渡层叠下去...

比如：

```css
/* bad */
div div div {
  ...
}

/* good */
.target {
  ...
}
```

后者的匹配效率是远远大于前者的。


### 预解析

主解析器在遇见 script 标签时会暂停解析，并将控制权交给 js 引擎。

此时其他线程会继续去解析文档的剩余部分，**找出并加载需要通过网络加载的其他资源**。这样就可以使网络资源并行加载，从而提高整体速度，这称之为 **预解析**。

预解析是不会去修改 DOM 树的，DOM 树的构建仅仅由主解析器进行。

## 布局树（渲染树）

> WebKit 中称布局树为 **RenderTree** 而 Blink 则称之为 **LayoutTree**。

从文章开始的流程图那里可以看出，DOM 树与 CSSOM 树共同构建成了一个新的树：布局树。

![render-tree-construction.png-34.1kB][11]

### 布局对象（渲染对象）

> WebKit 将布局对象称之为 `RenderObject`，而在 Blink 中则称之为 `LayoutObject`。

同 DOM 与 DOM 树的关系一样，布局树也同样是由 **布局对象** 按一定的结构构建而来。

每一个布局对象都代表了一个矩形的区域，通常对应于相关节点的 CSS 框，它包含诸如宽度、高度和位置等几何信息。布局对象的类型会受到节点相关的 display 属性所影响。

一个元素的样式为 `display: none;`，另一个元素的样式为 `visibility: hidden;`，有什么区别呢？最直观的就是 `visibility: hidden;` 虽然隐藏了，但是还占据着空间，而 `display: none;` 则消失的更彻底，在页面上找不见一点痕迹。那么对于布局对象来说，这两个又有什么区别呢？下面是一段 Blink 代码，里面描述了根据 display 属性对于布局对象的影响。

```cpp
// LayoutObject.cpp

LayoutObject* LayoutObject::createObject(Element* element, const ComputedStyle& style)
{
    //...
    //...

    switch (style.display()) {
    case NONE:
        return nullptr;
    case INLINE:
        return new LayoutInline(element);
    case BLOCK:
    case INLINE_BLOCK:
        return new LayoutBlockFlow(element);
    case LIST_ITEM:
        return new LayoutListItem(element);
    case TABLE:
    case INLINE_TABLE:
        return new LayoutTable(element);
    case TABLE_ROW_GROUP:
    case TABLE_HEADER_GROUP:
    case TABLE_FOOTER_GROUP:
        return new LayoutTableSection(element);
    case TABLE_ROW:
        return new LayoutTableRow(element);
    case TABLE_COLUMN_GROUP:
    case TABLE_COLUMN:
        return new LayoutTableCol(element);
    case TABLE_CELL:
        return new LayoutTableCell(element);
    case TABLE_CAPTION:
        return new LayoutTableCaption(element);
    case BOX:
    case INLINE_BOX:
        return new LayoutDeprecatedFlexibleBox(*element);
    case FLEX:
    case INLINE_FLEX:
        return new LayoutFlexibleBox(element);
    case GRID:
    case INLINE_GRID:
        return new LayoutGrid(element);
    }
    ASSERT_NOT_REACHED();
    return nullptr;
}
```

从代码中我们可以看到，当 `display: none`  时是不创建布局对象的。而 `visibility: hidden` 时只要其 display 的属性不是 none，依然是会创建对象。


### 构建布局树的流程

1. **从 DOM 树的根节点开始遍历每个可见节点**
    - 某些不可见节点（如 script 标签, meta  标签等）不需要渲染在页面中，因此会被忽略。
    - 设置了 `display: none` 样式属性的节点也不会被添加在布局树上。

2. **对于每个可见节点，为其找到适配的 CSSOM 规则并应用它们。**
    - 解析样式和创建布局对象的过程称为 “Attachment” 。每个 DOM 节点都有一个 “attach” 方法。附加是同步进行的，将节点插入 DOM 树需要调用新的节点 “attach” 方法。

3. **连同其内容和计算的样式来构建一个布局对象**

4. **由于 DOM 树的影响，布局对象之间也构成了树状结构：布局树**
    - 布局树虽然和 DOM 有联系，但结构并不完全相同。

### 布局树 和 DOM 树的关系

**布局对象是和 DOM 元素相对应的，但并非一一对应。非可视化的 DOM 元素不会插入布局树中。**例如“head”元素。如果元素的 display 属性值为“none”，那么也不会显示在布局树中（但是 visibility 属性值为“hidden”的元素仍会显示）。

对于一些特殊的 HTML 元素，如 select, video 等，它们往往是具有复杂结构的元素，浏览器会通过添加 **shadow DOM** 的方式来为其添加多个可视化对象。

> 可在 Chrome DevTools 中 **Settings -> Preferences -> Elements -> Show user agent shadow DOM** 开启对 shadow DOM 的元素审查。

有一些呈现对象对应于 DOM 节点，**但在树中所在的位置与 DOM 节点不同。浮动定位和绝对定位的元素就是这样，**它们处于正常的流程之外，放置在树中的其他地方，并映射到真正的框架，而放在原位的是占位框架。

### 布局树一定要等到 DOM 树构建完毕才开始构建吗？

先看下面一个示例：

有三个不同颜色的 div，red 和 yellow 在外部脚本前，blue 则处于外部脚本之后。

```html
<div class="d red"></div>
<div class="d yellow"></div>
<!-- 引入一个外部脚本 -->
<script src="/static/demo.js"></script>
<div class="d blue"></div>
```

在 Chrome DevTools 中模拟慢网环境以方便观察具体现象。

![js-load-before.png-195.4kB][12]
上图是浏览器正在请求 JS 文件的时刻。可以清楚的看到在 script 标签之前的 red 和 yellow 两个方块都被渲染出来的了。但是在其之后的 blue 没有被渲染出来。

![js-load-after.png-172.5kB][13]
当 JS 文件加载完成并执行完毕后，blue 随即也被渲染出来。

再看第二个示例：

还是三个不同颜色的 div，但是这一次换成内联脚本。为了方便查看效果，我们在脚本中写了一个 5 秒的死循环。

```html
<div class="d red"></div>
<div class="d yellow"></div>
<script>
  // 5 秒死循环
  console.log('begin');
  var time = new Date().getTime();
  while (new Date().getTime() - time < 1000 * 5) {
  }
  console.log('end');
</script>
<div class="d blue"></div>
```

在 Chrome DevTools 查看效果

![script-begin.png-162.7kB][14]

在控制台中打印出了 `begin` 证明 JS 脚本开始执行了。但是可以观察到页面上目前还是一片空白

![script-end.png-181.2kB][15]

随着控制台中打印出了 `end`，即 JS 脚本执行完毕。页面上开始渲染出了全部的 DOM 元素。

因此可以得出以下结论：

**布局树会在 HTML 解析至第一个外部脚本的时候开始构建。**

## 结语

本文主要介绍了 HTML 解析与相关树构建（包括 DOM 树，CSSOM 树以及布局树），这几个步骤对于页面首屏渲染是至关重要的，如果你想要在首屏渲染时间上有所优化，这里面也是大有文章可做，至于有哪些常见的措施来优化首屏加载速度呢？那就请继续阅读下一篇文章吧！

## 参考
- [Critical Rendering Path](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/)
- [How Browsers Work](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/)
- [HTML Living Standard - The HTML syntax](https://www.w3.org/TR/html5/syntax.html)
- [LayoutObject.cpp](https://chromium.googlesource.com/chromium/blink/+/master/Source/core/layout/LayoutObject.cpp)
- [LayoutView.cpp](https://chromium.googlesource.com/chromium/blink/+/master/Source/core/layout/LayoutView.cpp)

  [1]: http://static.zybuluo.com/pspgbhu/x7fd0pvcydddz0v8vc37uuza/rendering-process.png
  [2]: http://static.zybuluo.com/pspgbhu/9mi6j7osg01apgzjc0298giv/parsing-model-overview%20%281%29.png
  [3]: http://static.zybuluo.com/pspgbhu/15hbugs0m5ret8oebay7s7dg/cssom-construction.png
  [4]: http://static.zybuluo.com/pspgbhu/mpktzkj3x8bgkjc3shgbovro/cssom-tree.png
  [5]: http://static.zybuluo.com/pspgbhu/94y07sfhf3qf8jsbzx1hco80/34839297-1d67a892-f73c-11e7-9477-97fc10b15745.png
  [6]: http://static.zybuluo.com/pspgbhu/i5oob324pu3mzb57ptobkn7m/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-01-20%20%E4%B8%8B%E5%8D%887.30.18.png
  [7]: http://static.zybuluo.com/pspgbhu/s66hxejrr60bggj0az2fkzfy/frame-full.jpg
  [8]: http://static.zybuluo.com/pspgbhu/s66hxejrr60bggj0az2fkzfy/frame-full.jpg
  [9]: http://static.zybuluo.com/pspgbhu/u9q94e2djmlinbbunayjbfiw/frame-no-layout.jpg
  [10]: http://static.zybuluo.com/pspgbhu/u20r4qqthi76qxdu5nc1nlq0/frame-no-layout-paint.jpg
  [11]: http://static.zybuluo.com/pspgbhu/3r75tben2r73edhbtn5aei4r/render-tree-construction.png
  [12]: http://static.zybuluo.com/pspgbhu/zfi9oetjccvj0s0idf7z7s36/js-load-before.png
  [13]: http://static.zybuluo.com/pspgbhu/yrgu4r1em6971afn4hygxcxq/js-load-after.png
  [14]: http://static.zybuluo.com/pspgbhu/1n12h5txsrsqv6pgf7bvfg3p/script-begin.png
  [15]: http://static.zybuluo.com/pspgbhu/b4syzxci7uzeqbnmbdha1ytw/script-end.png
