---
title: Egret 个人实践
comments: true
date: 2017-12-24 02:10:11
categories: GameEngine
tags: Egret
img:
---


## 1. UI 与逻辑分离

在使用 Egret 做游戏的时候，由于 UI 样式与游戏逻辑都需要用 TS 代码来描述，因此做到样式与逻辑相分离，是每一个项目的最基本的要求。

### 1.1 官方推荐的解决方案 - EUI 库

EUI 是 Egret 的一个拓展库，在 Egret 官网中是这么介绍的：

> EUI是一套基于Egret核心显示列表的UI扩展库，它封装了大量的常用UI组件，能够满足大部分的交互界面需求，即使更加复杂的组件需求，您也可以基于EUI已有组件进行组合或扩展，从而快速实现需求。EUI里可以使用 EXML 来开发应用界面，标签式的语法更加适合 UI 开发， EXML 开发可以做到 UI 与逻辑代码的分离，更利于团队协作和版本迭代。

很明显，EUI 生来就是为了描述 UI 界面的。

在经过简单的体验之后，简单的总结了几个 EUI 的优点和缺点

#### 1.1.1 EUI 库的优势
1. XML 类语言在描述 UI 界面方面有天然的优势。
2. 数据绑定。
3. 处理布局或者逻辑复杂的 UI 界面时更具备优势。
4. 皮肤分离机制

![皮肤分离机制](http://cdn.dev.egret.com/egret-docs/extension/EUI/skin/separate/55cdcff23152f.png)
> 可能会有人比较担心皮肤分离后的性能，一个组件拆分了成了两个，不会增加嵌套层级吗？答案是不会，因为这里的皮肤并不是显示对象。您可以把它理解为一个数据对象，存储了初始化显示列表和外观需要的特定数据。将皮肤附加到逻辑组件上的过程就是对逻辑组件应用一些列外观创建的初始化操作。

#### 1.1.2 EUI 库的劣势
1. 其本身文件大小约 300kb。
2. 项目越复杂，就越适合用 EUI 库，但是在项目本身非常简单时，就显得有些不必要。

在综合考虑了 EUI 的优劣势之后，我在项目中选择了弃用 EUI，库文件体积过大是一个因素，因为项目本来就是做一个简单的小游戏，EUI 起到的作用非常有限是另一个因素。


### 1.2 通过原生代码来实现 UI 与样式分离

当然，在不借助 EUI 的情况下，我们也可以很简单的实现 UI 与样式相分离。

我们将容器定义成以下两种容器：

- UI 容器：只描述 UI 界面，不进行逻辑处理
- 逻辑容器：只处理逻辑，不描述界面 UI

同时，我们在目录结构和文件命名上也要区分这两种容器。对于 UI 组件，我本人习惯以 UI 两个字母结尾，如 `GameUI.ts`，这样可以很清楚的区分 ”UI 组件“和”逻辑组件“。文件存放的目录上也有区分，具体如何区分会在下面”开发目录结构“一节写到。

#### 1.2.1 UI 容器

UI 组件负责描述 UI 界面，与此同时，还需要暴露出一些属性，以方便逻辑组件获取到页面上的一些 UI 实例。

比如说一个页面上有一个按钮和一张图片，点击按钮之后会隐藏图片。这样 UI 组件的任务就是绘制好按钮和图片的大小和位置，与此同时向外暴露出按钮和图片的实例，方便逻辑组件调用。

```
class GameUI extends egret.Sprite {
    // 对外暴露出按钮和图片的实例
    public btn: egret.Bitmap;
    public image: egret.Bitmap;

    constructor() {
        super();
        // 添加按钮实例
        this.addButton();
        // 添加图片实例
        this.addImage();
    }

    private addButton(): void {
        // 绘制按钮 UI
        this.btn = new egret.Bitmap(RES.getRes('btn_png'));
        this.btn.width = 100;
        this.btn.height = 100;
        this.addChild(this.btn);
    }

    private addImage(): void {
        // 绘制目标图片
        this.image = new egret.Bitmap(RES.getRes('img'));
        this.image.width = 200;
        this.image.height = 200;
        this.image.x = 200;
        this.image.y = 200;
        this.addChild(this.image);
    }
}

```

#### 1.2.2 逻辑容器

逻辑组件负责处理游戏逻辑，**通常逻辑组件需要将 UI 容器作为子组件引入**。

还是上面那个例子，在 UI 容器描述完页面 UI 后，我们在逻辑容器中引入 UI 容器作为子组件，并最终将整个逻辑容器添加在舞台上。

```
class GameScene extends egret.Sprite {
    private gameUI: GameUI;
    private btn: egret.Bitmap;
    private image: egret.Bitmap;

    constructor() {
        super();
        this.gameUI = new GameUI();
        // 从 UI 容器上拿到按钮和图片的实例
        this.btn = this.gameUI.btn;
        this.image = this.gameUI.image;

        // 绘制 UI
        this.addChild(this.gameUI);

        // 绑定事件
        this.bindEvent();
    }

    private bindEvent(): void {
        // 按钮监听点击事件
        this.btn.touchEnabled = true;
        this.btn.addEventListener(egret.TouchEvent.TOUCH_TAP, btnTouchHandler, this);
    }

    private btnTouchHandler(): void {
        // 隐藏图片
        this.image.visible = false;
        // 移除按钮点击事件
        this.btn.addEventListener(egret.TouchEvent.TOUCH_TAP, btnTouchHandler, this);
        this.btn.touchEnabled = false;
    }
}

```

## 2. 开发目录结构

开发目录，即 Egret 标准工程下的 `src` 目录。 清晰的目录结构同清晰的代码逻辑一样重要。下面是我所实践的开发目录结构。

```
# 游戏一共两个场景，开始场景和游戏主场景

src
├── Config.ts           # 全局配置
├── GameEvent.ts        # 统一管理自定义事件名称常量
├── Main.ts             # 游戏舞台
├── components          # 一些通用的UI组件和逻辑组件
│   ├── MonsterUI.ts    # 怪物的UI组件
│   ├── Monster.ts      # 怪物的逻辑组件
│   └── Player.ts       # UI及其简单就不需要分离UI了
├── scenes              # 游戏场景
│   ├── BeginScene.ts   # 开始场景
│   └── GameScene.ts    # 游戏主场景
└── uis                 # UI 组件
    ├── BeginRuleUI.ts  # 开始场景中的一个UI组件
    ├── BeginUI.ts      # 开始场景主UI容器
    ├── GameUI.ts       # 游戏场景主UI容器
```

## 3. 优化

## 3.1 对象池

创建对象时，将对象创建在循环外部并在循环内反复重用它。并非所有对象总是能够这么做，但在许多情形下此技术很有帮助。Egret 官方给出了一个 [Egret 对象池的实现](https://github.com/egret-labs/egret-game-library/blob/master/src/game/ObjectPool.ts)。

或者说我们根据自己的业务实现一个简单的对象池

```typescript
// 一个 Monster 实例的对象池
class MonsterPool {
    private pool: Array<Monster>;

    constructor() {
        this.pool = [];
    }

    // 从对象池中取用对象
    private getMonster(): Monster {
        if (this.pool.length) {  // 如果池子里有对象
            return this.pool.shift();  // 取一个对象
        } else {        // 如果对象池中没有对象
            return new Monster();   // new 一个实例
        }
    }

    // 回收对象
    private recoverMonster(monster: Monster) {
        // 给对象池的对象数量设置一个合理的上限
        // 避免储存过量的对象，造成内存溢出
        if (this.pool.length > 30) return;
        // 实例中最好提供一个 reset 方法来重置对象实例的属性
        monster.reset();
        this.pool.push(monster);
    }
}u
```

其实最关键的不是对象池是如何实现的，而是需要具备复用对象的思想。


### 3.2 Loading 页面

用原生 DOM 来写 Loading，而不是用 Egret 创建 Loading 页面。

因为只有在 Egret 及其相关依赖 JS 文件加载完毕并且开始执行的时候用 Egret 描写的页面才能展示出来。而且通常情况下，Egret 及其 JS 依赖的文件大小在 300kb 左右，这已经是一个不小的数目了。

因此使用原生 DOM 来写 Loading 页面可以尽快的呈现给用户。


### 3.2 通用背景

如果游戏中的背景是在任何场景都通用的，那么我们不妨把这个通用背景直接渲染在 DOM 上而不是在 canvas 中。
