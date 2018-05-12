---
title: 使用正则表达式来匹配不是以 XXX 结尾的字符串
comments: true
date: 2018-05-13 02:00:58
categories: JavaScript
tags: RegExp
img: http://static.zybuluo.com/pspgbhu/q39xwwbx7xpk0s6jbe4f6er5/image.svg
---

这个问题是前两天在项目中拓展延伸出来的，我最开始认为像这种使用正则来匹配 “非 XXX 结尾” 字符串的需求应该还算是满常见的，但是自己在实现的时候始终有一些问题，没有使得这个正则很通用。后来想着去网上搜搜看吧，也没有找到满意的答案。好在最后经过高人指点，还是解决了这个问题。

## 测试正则表达式的网站

1. [regexper](https://regexper.com/)

![regexper](http://static.zybuluo.com/pspgbhu/xzif1mcmaarbohjilvhmwl3l/regexper.png)

这个网站的牛逼之处在于，它以图形化的形式将你的正则来展示出来。这样不仅对于我们写正则，甚至对于理解正则都有着很大的帮助。


2. [regextester](https://www.regextester.com/) （需要科学上网）

![regextester](http://static.zybuluo.com/pspgbhu/qbh569wf3u9shlkip7wmtk3h/regextester.png)

它可以让你一边写正则一边对字符串进行实时匹配。

## 正向否定预查

>正向否定预查(negative assert)，在任何不匹配pattern的字符串开始处匹配查找字符串。这是一个非获取匹配，也就是说，该匹配不需要获取供以后使用。例如"Windows(?!95|98|NT|2000)"能匹配"Windows3.1"中的"Windows"，但不能匹配"Windows2000"中的"Windows"。预查不消耗字符，也就是说，在一个匹配发生后，在最后一次匹配之后立即开始下一次匹配的搜索，而不是从包含预查的字符之后开始。
>
> 摘自：正则表达式 - 元字符 - 菜鸟教程

当时有这个需求的时候，我第一反应就是使用正向否定预查。后来证明这个想法倒也没错，只不过在具体的使用上确实犯了一些错误，导致多走了一些弯路。

> 需求： node_modules 目录下所有以 css 或 less 结尾的文件返回 false，其余返回 true。用 `RegExp.prototype.test` 来实现

我第一版的正则表达式是下面这样的。正则前半部分 `[\\/]node_modules[\\/]` 没什么好看的，主要关注后半部分。


```js
// 这是一个错误示例
var reg = new RegExp(/[\\/]node_modules[\\/].*?(?!css|less)$/);
reg.test('/node_modules/grrroop/demo.css'); // true
reg.exec('/node_modules/grrroop/demo.css');
// ["/node_modules/grrroop/demo.css", index: 0, input: "/node_modules/grrroop/demo.css", groups: undefined]
```

后来发现无论字符结尾，这个正则都会返回 true。后来使用 `RegExp.prototype.exec` 方法看了一下，是因为 `.*` 的组合直接一路匹配到字符串结尾了，而我在其之后加 `?` 来让其变得非贪婪好像也没有起到预期的效果。

经过网上查阅资料得知，正向否定预查前不能是通配符或者留空。

之后又找了个大神指导了一番，终于发现了正确的姿势：

```js
// 正确示例
var reg = new RegExp(/[\\/]node_modules[\\/](?!.*?(css|less)$)/);
reg.test('/node_modules/grrroop/demo.css'); // false
```

后面 `(?!.*?(css|less)$)` 这样子写就对了，这里的正向否定，就相当于对 `.*?(css|less)$` 的结果取了一个反。

## 更通用些

业务问题解决了，我们就要开始去抽象这个问题，使其变得更加的通用。

经过抽象后的问题就变成了：**使用正则表达式来匹配不是以 XXX 结尾的字符串**

```js
/^(?!.*?XXX$)/
```

记住重要的一点：正向否定预查前不能是通配符或者留空。所以我在表达式前加上了 `^` 限定符。

只要是包含 **不是以 XXX 结尾** 都可以用这种方法解决：

```js
/^Begin(?!.*?NotEnd$)/ // 以 Begin 开头，但不已 NotEnd 结尾

/keyword(?!.*?NotEnd$)/ // 包含 keyword，但不已 NotEnd 结尾
```
