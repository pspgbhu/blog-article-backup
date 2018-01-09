---
title: DOM Level 3 Events VS DOM Level 2 Events
comments: true
date: 2018-01-07 17:33:31
categories: DOM
tags: DOM-Events
img: https://www.w3.org/TR/DOM-Level-3-Events/images/eventflow.svg
---

写这篇博客的原因是《JavaScript 高级程序设计》（以下简称《高程》）中关于**事件**这一章有很多内容主要是以 **DOM2 级事件** 为标准来描述的，一些内容已经滞后于最新的 DOM3 标准了。因此本文就根据W3C现行草案来讲述一下最新的变更。

## 事件流

### DOM Level 2 Events 事件流
先看一下《高程》中的一个配图

![20161025100517307](https://user-images.githubusercontent.com/18444796/34526733-58efb3a0-f0de-11e7-9355-f804ae9de9eb.jpg)

《高程》书中是这么描述的：

> 捕获阶段事件对象会一直从 `Document` 开始向下传递到 `target` 目标元素的父元素上，然后从目标元素上开始进入冒泡阶段，一直向上传递到 `Document` 对象上。
>
> 在事件处理程序中，事件对象“处于目标阶段”也不是真正的处于该阶段，而是处于“冒泡阶段”。

然而在新标准下，并不是这个样子的。

### DOM Level 3 Events 级事件流

下图是 W3C 最新草案中附上的事件流示意图

![](https://www.w3.org/TR/DOM-Level-3-Events/images/eventflow.svg)

DOM 事件流中规定了事件流包含三个阶段：**事件捕获阶段(capture phase)**，**处于目标阶段(target phase)** 和 **事件冒泡阶段(bubble phase)**

当在页面中某个元素上触发事件时，就会先进入到 **事件捕获阶段**，该阶段会从 **window 对象开始** 一级一级沿着目标元素的祖先元素们向下捕获（DOM2 中规定事件应从 document 对象开始传播，但在 DOM3 中增加了 window 对象），直到到达触发事件的**目标元素的父元素为止**。然后在触发事件的目标元素上进入 **处于目标阶段**，而后进入**事件冒泡阶段**，从目标元素的**父元素开始**一直到传播到 window 对象上。

#### 事件流三个阶段的准确定义

> W3C 最新的 DOM Events 草案关于三个阶段的定义。
>
> - **The capture phase:** The event object propagates through the target’s ancestors from the Window to the target’s parent. This phase is also known as the capturing phase.
>
> - **The target phase:** The event object arrives at the event object’s event target. This phase is also known as the at-target phase. If the event type indicates that the event doesn’t bubble, then the event object will halt after completion of this phase.
>
> - **The bubble phase:** The event object propagates through the target’s ancestors in reverse order, starting with the target’s parent and ending with the Window. This phase is also known as the bubbling phase.

- **事件捕获阶段：**从 window 对象开始，事件对象会沿着目标元素的祖先们一直传播到目标元素的父元素上，这个阶段也被称之为事件捕获阶段。

- **处于目标阶段：**事件对象到达了事件对象中的 target 元素上。这个阶段被称之为处于目标阶段。如果事件类型指示事件不起泡，则在完成此阶段后，事件对象将停止传播。

- **事件冒泡阶段：**事件对象以相反的顺序传播到目标的祖先，从目标元素的父元素一直到 window 对象，这个阶段被称为事件冒泡阶段。

### 有变更的地方

1. 旧内容：~~规定 Document 为事件流顶层对象。~~
   **新内容：规定 Window 为事件流顶层对象。**

    在 DOM Level 2 Events 中明确规定了事件应从 document 对象开始传播，但是各大浏览器厂商纷纷将 window 对象也加入了事件流中。此次 DOM3 Level 中尊重了这个既成事实，将 window 对象加入了 Event Flow 中。

2. 旧内容：~~冒泡阶段是从目标元素开始向上传播的，同时在事件处理程序中事件对象“处于目标阶段”时（即 Event.eventPhase === 2）也不是真正的处于该阶段，而是处于“冒泡阶段”。~~
   **新内容：冒泡阶段是从目标的父元素开始向上传播的，同时在事件处理程序中事件对象“处于目标阶段”时（即 Event.eventPhase === 2）便是真正的处于目标阶段了。**

3. 旧内容：~~DOM2 Level 中没有明确规定多个事件触发的顺序。~~
   **新内容：现在多个事件触发的顺序已经在 DOM3 Level 中明确规定了。**

   可通过 “event order” 关键字在页面中搜索查看最新标准 [W3C UI Events 现行草案](https://www.w3.org/TR/DOM-Level-3-Events/#events-focusevent-event-order)

## 事件处理程序

### DOM Level 2 Events 中的规定

DOM2 级事件 中定义了两个方法，用于处理指定和删除事件处理程序的操作：`addEventLisenter()` 和 `removeEventListener()`。所有的 DOM 节点都包含这两个方法。

这两个函数都接受一个布尔值作为第三个参数，用来控制是在捕获阶段或者冒泡阶段来调起事件处理程序。

### DOM Level 3 Events 中更新的地方

DOM3 中主要是拓展了 `addEventListener()` 中的第三个函数，现在不仅仅能够接受一个布尔值作为第三个参数，而且还能够接受一个 option 对象作为第三个参数，其可选的参数有：

- **capture: Boolean**
表示 listener 会在该类型的事件捕获阶段传播到该 EventTarget 时触发。

- **once: Boolean**
表示 listener 在添加之后最多只调用一次。如果是 true， listener 会在其被调用之后自动移除。

- **passive: Boolean**
表示 listener 永远不会调用 preventDefault()。如果 listener 仍然调用了这个函数，客户端将会忽略它并抛出一个控制台警告。
mozSystemGroup: 只能在 XBL 或者是 Firefox’ chrome 使用，这是个 Boolean，表示 listener 被添加到 system group。

具体可参考 [MDN addEventListener](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/addEventListener)


## 事件对象

事件对象也有许多的改动，就挑几个主要的来说一下

### 新的属性和方法

#### 属性 [defaultPrevented](https://dom.spec.whatwg.org/#dom-event-defaultprevented)

当改事件的默认行为被 `preventDefault()` 方法取消时，返回 false，否则返回 true。

```javascript
var mydiv = document.getElementById('mydiv');
mydiv.addEventListener('touchmove', function(e) {
    e.preventDefalut();
});
mydiv.addEventListener('touchmove', function(e) {
    // 当在 mydiv 上触发 touchmove 事件时，
    // 总是可以打印出 true
    console.log(e.defaultPrevented);
});
```
同时还需要注意，**一个元素使用 `addEventListe` 绑定了多个事件监听程序时，监听程序按照注册的先后顺序依次调用。**所以说如果把上例中两个事件监听程序顺序颠倒一下，打印出来的值就会变成 false 了。

*注意：当事件对象 `Event.cancelable` 的值为 false，即不可以取消默认事件时，即使调用了 `preventDefault()`，`defaultPrevented` 也总是返回 false*

#### 方法 [stopImmediatePropagation()](https://dom.spec.whatwg.org/#dom-event-stopimmediatepropagation)

这个方法和 `stopPropagation()` 长得很相似，但作用还是有所不同：

- `stopPropagation()` 方法可以阻止事件的传递，使其之后父一级的事件监听程序无法被调起。
- `stopImmediatePropagation()` 不仅仅可以其之后父一级的事件监听程序被调用，同时还可以阻止同级其他事件监听程序被调用。

下面有一个例子可以看看直观的感受到两者的区别。

```javascript
var mydiv = document.getElementById('mydiv');

mydiv.addEventListener('click', function(e) {
    e.stopPropagation();
    console.log(1);
});
// 同级的事件监听程序
mydiv.addEventListener('click', function(e) {
    console.log(2);
});
// 父一级的事件监听程序
document.body.addEventListener('click', function(e) {
    console.log(3);
});
// 触发 click, 将会打印出：
// 1
// 2
```

```javascript
var mydiv = document.getElementById('mydiv');

mydiv.addEventListener('click', function(e) {
    e.stopImmediatePropagation();
    console.log(1);
});
// 同级的事件监听程序
mydiv.addEventListener('click', function(e) {
    console.log(2);
});
// 父一级的事件监听程序
document.body.addEventListener('click', function(e) {
    console.log(3);
});
// 触发 click, 将只会打印出：
// 1
```

### 其他

- DOM Level 3 Events 规定 type 属性（即 addEventListener 的第一个参数）是区分大小写的。而 DOM Level 2 Events 是规定不区分大小写的。
- `EventTarget.dispatchEvent()` 有[修改](https://dom.spec.whatwg.org/#dom-eventtarget-dispatchevent)。
- `MouseEvent interface` 下新增了 `getModifierState()` 方法。
- 键盘事件方面有大量更新。

## 写在最后

值得一提的是，我本以为 W3C 的标准草案会非常的晦涩难懂，但是没有想到是草案不仅清晰易懂，而且图文并茂，同时标准草案有非常高的权威性，不用花精力去甄别真伪性，因此阅读技术相关标准草案是一种非常快速而且精准的学习方式。

## Resource

- [W3C Working Draft DOM-Level-3-Events](https://www.w3.org/TR/DOM-Level-3-Events/)
- [Interface EventTarget](https://dom.spec.whatwg.org/#interface-eventtarget)
- 《javascript 高级程序设计》
- [stopPropagation vs. stopImmediatePropagation](https://stackoverflow.com/questions/5299740/stoppropagation-vs-stopimmediatepropagation)
