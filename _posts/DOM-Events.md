---
title: 梳理 DOM 事件相关知识
comments: true
date: 2017-12-03 17:36:45
categories: JavaScript
tags: DOM
toc: true
---

> 本文尽可能的不啰嗦，也尽可能的不放过每一个重点。

浏览器事件系统相对比较复杂，因此本文只梳理其中几个最重要的概念。

## 事件流
事件流描述的是从页面中接受事件的顺序。 IE 和 Netscape 团队提出了不同的事件流概念：

- IE 提出的是 **事件冒泡流**
- Netscapte 提出的是 **事件捕获流**

### DOM 事件流
我们现在常说的 “DOM 事件流” 是在 **DOM2** 中规范并标准化，在 **DOM3** 进一步的拓展和完善。

*下图是 DOM3 事件流 示意图*
![event flow](https://www.w3.org/TR/DOM-Level-3-Events/images/eventflow.svg)

DOM 事件流中规定了事件流包含三个阶段：**事件捕获阶段(capture phase)**，**处于目标阶段(target phase)** 和 **事件冒泡阶段(bubble phase)**

当在页面中某个元素上触发事件时，就会先进入到 **事件捕获阶段**，该阶段会从 **window 对象开始** 一级一级沿着目标元素的祖先元素们向下捕获（DOM2 中规定事件应从 document 对象开始传播，但在 DOM3 中增加了 window 对象），直到到达触发事件的目标元素为止。然后在触发事件的目标元素上进入 **处于目标阶段**，而后进入**事件冒泡阶段**，从目标元素的**父元素开始**一直到传播到 window 对象上。


### 三个阶段的准确定义

> W3C 最新的 DOM Events 草案关于三个阶段的定义。

> - **The capture phase**: The event object propagates through the target’s ancestors from the Window to the target’s parent. This phase is also known as the capturing phase.

> - **The target phase**: The event object arrives at the event object’s event target. This phase is also known as the at-target phase. If the event type indicates that the event doesn’t bubble, then the event object will halt after completion of this phase.

> - **The bubble phase**: The event object propagates through the target’s ancestors in reverse order, starting with the target’s parent and ending with the Window. This phase is also known as the bubbling phase.

- **事件捕获阶段**：从 window 对象开始，事件对象会沿着目标元素的祖先们一直传播到目标元素的父元素上，这个阶段也被称之为事件捕获阶段。
- **处于目标阶段**：事件对象到达了事件对象中的 target 元素上。这个阶段被称之为处于目标阶段。如果事件类型指示事件不起泡，则在完成此阶段后，事件对象将停止传播。
- **事件冒泡阶段**：事件对象以相反的顺序传播到目标的祖先，从目标元素的父元素一直到 window 对象，这个阶段被称为事件冒泡阶段。

### DOM3 事件流中的新规定

#### 1. 明确定义了 Event Order
在 DOM2 Level 中并没有明确定义事件触发顺序，现在这些都在 DOM3 Level 中明确定义了。

可通过 “event order” 关键字在页面中搜索查看最新标准 [focusevent-event-order](https://www.w3.org/TR/DOM-Level-3-Events/#events-focusevent-event-order)

#### 2. 在事件流中增加了 window 对象
在 DOM2 Level 中明确规定了事件应从 document 对象开始传播，但是各大浏览器厂商纷纷将 window 对象也加入了事件流中。此次 DOM3 Level 中尊重了这个既成事实，将 window 对象加入了 Event Flow 中。

---

## 事件处理程序
事件就是用户或浏览器自身执行的某种动作。而相应某个事件的函数就叫做**事件处理程序（Event Handler）**或**事件监听器（Event Listener）**

### DOM0 级事件处理程序
```javascript
var btn = document.getElementById('btn');
btn.onclick = function () {
    console.log(this);  // btn element
};
```
上述代码中的 onclick 函数就是一个 DOM0 级的事件处理程序。
对于一个元素调用类似的 on + event 方法，就是 DOM0 级的事件处理程序。

**事件处理程序是在元素的作用域中运行的**，因此函数中的 this 会指向当前的元素。

### DOM2 级事件处理程序
DOM2 级事件 中定义了两个方法，用于处理指定和删除事件处理程序的操作：`addEventLisenter()` 和 `removeEventListener()`。所有的 DOM 节点都包含这两个方法。

#### addEventListenter()
该方法接受三个参数：

1. 要处理的事件名
2. 处理事件程序的函数
3. 布尔值，true 表示在捕获阶段调用事件处理程序，false 表示在冒泡阶段调用处理程序，默认 false。

与 DOM0 级事件相同，**事件处理程序是在元素的作用域中运行的**。相比于 DOM0 级事件处理程序最大的一个优势就是，可以为一个元素添加**多个事件处理程序**。

当添加了多个事件处理程序函数的时候，**函数会按照他们添加到的顺序触发**。

#### removeEventListenter()
通过 `addEventListener()` 添加的事件处理程序只能通过 `removeEventListener()` 来移除。移除时传入的参数与添加处理程序时使用的参数相同。这也意味着**通过 `addEventListener()` 添加的匿名函数将无法被移除**

### DOM3 级事件处理程序
DOM3 中主要是拓展了 `addEventListener()` 中的第三个函数，现在不仅仅能够接受一个布尔值作为第三个参数，而且还能够接受一个 option 对象作为第三个参数，其可选的参数有：

- **capture**:  Boolean
    表示 listener 会在该类型的事件捕获阶段传播到该 EventTarget 时触发。

- **once**:  Boolean
    表示 listener 在添加之后最多只调用一次。如果是 true， listener 会在其被调用之后自动移除。
- **passive**: Boolean
    表示 listener 永远不会调用 preventDefault()。如果 listener 仍然调用了这个函数，客户端将会忽略它并抛出一个控制台警告。
 mozSystemGroup: 只能在 XBL 或者是 Firefox' chrome 使用，这是个 Boolean，表示 listener 被添加到 system group。

具体可参考 [MDN addEventListener](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/addEventListener#规范)

---

## 事件对象
在 DOM 上触发某个事件时，会产生一个事件对象，这个对象中包含

触发的事件类型不一样，可用的属性和方法也不相同。不过，所有的事件都会有下列成员：

| 属性/方法 | 类型 | 读/写 | 说明 |
| --------- | ---- | ---- | ---- |
| bubbles | Boolean | r | 表明事件是否冒泡 |
| cancelable | Boolean | r | 表明是否可以取消事件的默认行为 |
| **currentTarget** | Element | r | 其事件处理程序当前正在处理事件的那个元素 |
| detail | Intrger | r | 与事件相关的细节
| eventPhase | Integer | r | 调用事件处理程序的阶段：1表示捕获阶段，2表示处于目标，3表示冒泡。这里的2其实是有问题的，下面会详解。
| preventDefault() | Function | r | 取消事件的默认行为。如果 cancelable 是 true，则可以使用此方法
| stopImmediatePropagation() | Function | r | 取消事件的进一步捕获或冒泡，同时阻止任何事件处理程序被调用（DOM3 中新增）|
| stopPropagation() | Function | r | 取消事件的进一步捕获或冒泡。如果 bubbles 为 true，则可以使用这个方法 |
| **target** | Element | r | 事件的目标 |
| trusted | Boolean | r | 为 true 表示事件是浏览器生成的。为 false 表示事件是由开发人员通过 JavaScript 创建的（DOM3 新增）|
| type | String | r | 被触发的事件类型 |
| view | AbstractView | r | 与事件关联的抽象视图。等同于发生事件的 window 对象 |

**以下几点需要牢记**：

- 在事件处理程序内部，对象 this 始终等于 currentTarget 的值。
- target 等于是实际触发事件的元素。
- IE 的事件对象下会有一个 `srcElement` 等同于 `target`，这个属性在 chrome 中仍然存在，但在 firefox 中不支持，因此请用 `target` 来替代 `srcElemnt`
- 目前只有**冒泡阶段**和**捕获阶段**能够调用事件处理程序。**目标阶段**并不能调用事件处理程序。
- 事件处理程序函数默认是在**冒泡阶段**被调用的，因此当 `event.eventPhase` 值为 2 时，并不是表明其真正处于目标阶段，**而是处于冒泡阶段**。


**事件对象下更多属性请参考：**

- [Interface Event](https://dom.spec.whatwg.org/#interface-event)
- [DOM-Level-3-Events Interface](https://www.w3.org/TR/DOM-Level-3-Events/#event-types)
[](https://dom.spec.whatwg.org/#interface-event)


---

## References
- [W3C Working Draft DOM-Level-3-Events](https://www.w3.org/TR/DOM-Level-3-Events/)
- [Interface EventTarget](https://dom.spec.whatwg.org/#interface-eventtarget)
- [MDN addEventListener](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener)
- 《javascript 高级程序设计》
