---
title: 用 Egret 引擎制作 HTML5 游戏
comments: true
date: 2017-12-09 10:58:26
categories: GameEngine
tags: Egret
img:
---

> 2017-12-23 更新：通过自定义事件来帮助场景间的切换，监听 `egret.Event.ADDED` 来在监听容器被添加进显示列表的事件。


近期有机会接触到了 HTML5 游戏引擎，虽然之前也做过一些移动端的小游戏，但大多是通过 DOM 来实现的，这次有机会接触到给予 canvas 的游戏引擎也是感触颇多，同时也发现网上关于 Egret 起步的文章也很少，因此便打算将自己的起步经验记录下来，希望能对后来人起到一点点的帮助。

## 安装 Egret 引擎

安装引擎这个步骤其实官网教程讲的也很详细 [Egret 安装与部署](http://developer.egret.com/cn/2d/projectConfig/installation)，在 [Egret 官网下载](https://www.egret.com/products/engine.html) 下载 Egret Engine，下载并安装之后会发现其本身是一个 Launcher，具备启动和管理 Egret 引擎和插件的一个东西。

然后只用在 Launcher 中安装最新的引擎即可，其他的我们暂时先不管。
![egret launcher](/images/egret_launcher.jpg)

## 创建一个新的项目

### 1. 通过命令行快速生成项目模板
我们来使用 egret 命令行命令来创建一个新的项目。首先先打开本地命令行，运行如下代码来创建一个名为 demo 的项目：

```bash
egret create demo
```
这样我们就快速的生成了一个 demo 项目在本地。

### 2. 模板目录结构介绍
```
.
├── bin-debug               ## 编译后的本地开发文件目录
├── egretProperties.json    ## 项目配置文件
├── favicon.ico
├── index.html              ## 本地开发的 debug 版本 html 模板
├── libs                    ## 存放项目库文件
├── manifest.json           ## 项目所有所需的 JS 文件路径配置，引擎会自动修改，不需要手动管理
├── promise
├── resource                ## 项目资源管理目录，包括图片和 JSON
│   └── default.res.json    ## 资源管理配置文件，只有在本文件配置过的资源才能被正确的加载
├── src                     ## 主开发目录
├── template                ## publish 模板目录
│   ├── runtime
│   └── web
│       └── index.html      ## release 版本 html 模板
└── tsconfig.json           ## TypeScript 配置文件
```

### 3. 运行 demo

```
egret startserver -a
```

运行上述命令会在本地起一个服务来进行实时预览，后面的 `-a` 的参数可以自动检测本地文件变动，实时编译。

## 开始制作游戏

### 1. 修改运行配置

在 `index.html` 文件中的找到 class 类名为 `egret-player` 的 div 元素，通过修改其自定义属性来改变其运行配置，以下是全部的配置属性和其含义：

- data-entry-class：文件类名称。
- data-orientation：旋转模式。
- data-scale-mode：适配模式。
- data-frame-rate：帧频数。
- data-content-width：游戏内stage宽。
- data-content-height：游戏内stage高。
- data-show-pain-rect：是否显示脏矩形区域。
- data-multi-fingered：多指最大数量。
- data-show-fps：是否显示fps。
- data-show-log：是否显示egret.log的输出信息。
- data-show-fps-style：fps面板的样式。支持5种属性，x:0, y:0, size:30, textColor:0xffffff, bgAlpha:0.9

大多数配置是不用修改的，我们现在来改这几个属性：

1. data-scale-mode="exactFit"，这样子整个游戏舞台便会非等比缩放总是填满窗口。
2. data-frame-rate="60"，修改游戏默认渲染帧数为 60 帧每秒。


```html
<div style="margin: auto;width: 100%;height: 100%;" class="egret-player"
      data-entry-class="Main"
      data-orientation="auto"
      data-scale-mode="showAll"
      data-frame-rate="30"
      data-content-width="640"
      data-content-height="1136"
      data-show-paint-rect="false"
      data-multi-fingered="2"
      data-show-fps="false" data-show-log="false"
      data-show-fps-style="x:0,y:0,size:12,textColor:0xffffff,bgAlpha:0.9">
</div>
```

### 2. 书写游戏主逻辑

写之前我们先要大概规划一下整个游戏一共分为多少个场景。我们整个游戏一共两个场景：
1. 一般所有的游戏开始都先有一个开场场景，我们这个也不例外。
2. 游戏交互的场景。

其次，还有一个贯穿始终的舞台背景图。

#### 2.1 开场

首先我们为整个游戏添加一个通用的背景图，这个图片将直接添加在根容器上。

##### 2.1.1 配置图片资源

先把 demo 自带的资源删除掉，即删除 `resource/images` 下面的文件，同时 `resource/config` 这个文件夹删除掉。然后将我们准备好的资源图片复制到 `resource/images/` 文件夹下，然后修改 `resource/default.res.json` 文件如下：
```json
{
	"groups":[
	{
		"keys":"bg,title",      ## 需要预加载进缓存的图片组，用逗号分隔
		"name":"preload"
	}],
	"resources":[
	{
		"name":"bg",	          ## 资源 name 值
		"type":"image",		      ## 资源类型
		"url":"assets/bg.png"		## 资源地址
	},
	{
		"name":"title",
		"type":"image",
		"url":"assets/title.png"
	}]
}
```
`resources` 字段用来下配置所有的资源，`groups` 下配置由单个资源组成的资源组。因为 `src/Main.ts` 中已经实现了对 preload 组下的资源文件预加载，所以这里我们把所有的资源都配置到 preload 资源组中。

##### 2.1.2 在根容器上添加图片

先删除 `src/Main.ts` 文件下的 `createGameScene()` 函数的全部内容。然后增加添加舞台背景的代码：
```javascript
private createGameScene() {
  const bg = new egret.Bitmap(RES.getRes('bg'));  // 新建一个位图对象，游戏背景图
  bg.width = this.stage.stageWidth;       // 将图片的宽度设置成舞台的宽度
  bg.height = this.stage.stageHeight;     // 将图片的高度设置成舞台的高度
  this.addChild(bg);      // 添加图片进根容器
}
```
ok，这样背景就添加好了

#### 2.2 创建开场场景 及 场景切换的正确姿势

在 `src` 目录下新建 `LayerBegin.ts` 文件，新建 `LayerBegin` 类来作为游戏的第一个场景。

**当 src 目录下的文件有变动的时候，manifest.json 文件会自动更新，不需要手动配置**

```javascript
// src/BeginLayer.ts
class LayerBegin extends egret.DisplayObjectContainer {

  constructor() {
    super();
    this.addEventListener(egret.Event.ADDED, this.init, this);
  }

  public init() {
    // 必要要移除这个，否则会造成死循环，具体原因下面会说明
    this.removeEventListener(egret.Event.ADDED, this.init, this)
    // 新建一个位图对象，游戏标题
    const title: egret.Bitmap = new egret.Bitmap(RES.getRes('title'));
    title.width = 549;
    title.height = 154;
    title.x = (this.stage.stageWidth - 549) / 2;   // 左右居中
    title.y = 300;
    this.addChild(title);

    // 当用户点击页面时，触发事件
    this.stage.addEventListener(egret.TouchEvent.TOUCH_TAP, handleTap, this);
  }

  public handleTap() {
    // 下发场景结束事件
    this.dispatchEventWith('SCENE_END');
    // 不要忘记卸载事件
    this.stage.removeEventListener(egret.TouchEvent.TOUCH_TAP, handleTap, this);
  }
}
```
现在就来说一下为什么必须要手动卸载这个事件。`egret.Event.ADDED` 事件不仅仅在容器被加入到显示列表中被调用，在其子组件被加入到显示列表中的时候，由于**事件冒泡机制**，也会导致该容器的 `ADDED` 事件被触发。

如果在上面调用 init 方法的时候不卸载 `ADDED` 事件，由于 init 方法中将 title 子组件加入到了显示列表中，因此事件会冒泡到该容器上，再次触发 `ADDED` 事件，形成死循环。


另一件需要注意的事情是，**其中 `this.stage` 在实例未被添加进舞台的时候，其值是 null。**

这样，我们第一个场景的类就开发完毕了，现在来将其实例化到舞台上。继续完成 `src/Main.ts` 中的 `createGameScene()` 函数。

```javascript
private createGameScene() {
  // 设置整体背景
  const bg: egret.Bitmap = new egret.Bitmap(RES.getRes('bg'));  // 新建一个位图对象，游戏背景图
  bg.width = this.stage.stageWidth;       // 将图片的宽度设置成舞台的宽度
  bg.height = this.stage.stageHeight;     // 将图片的高度设置成舞台的高度
  this.addChild(bg);      // 添加图片进根容器

  // 舞台第一幕
  let layerBegin: LayerBegin = new LayerBegin();    // 新建实例
  this.addChild(layerBegin);  // 添加进根容器
  // 监听第一幕谢幕事件，并切换第二幕
  layerBegin.addEventListene(egret.TouchEvent.TOUCH_TAP, toGameLayer, this);
}

toGameLayer() {
  // 不要忘记卸载事件
  layerBegin.removeEventListener(egret.TouchEvent.TOUCH_TAP, toGameLayer, this);
  // 从根容器中移除实例
  this.removeChild(layerBegin);
}
```

#### 2.3 创建游戏主场景

结束开始场景后，就进入到了我们游戏的主场景。今天我们来做这么一个游戏：

控制主人公左右移动来躲避从屏幕下方飞来的敌人，碰到敌人则 Game Over。

大概缕一下，我们需要创建这么几个类：

1. Player 类，既玩家所控制的元素。
2. Flyer 类，飞行的敌人的类。
5. 一个场景容器 LayerGame 类。

这里就不贴大段代码了，具体源码可参见 [这里](https://github.com/pspgbhu/egret-sample-game/tree/master/src)

主要就说一下几个重要功能的实现

##### 2.3.1 Player 随手指滑动而移动
通过 `this.stage.addEventListener` 方法来监听 touch 事件，然后来改变元素的 x 轴的位置。

```javascript
// src/Player.ts

  // 绑定触摸事件
  private bindEvent(): void {
      this.stage.addEventListener(egret.TouchEvent.TOUCH_BEGIN, this.handleTouchBegin, this);
      this.stage.addEventListener(egret.TouchEvent.TOUCH_END, this.handleTouchEnd, this);
  }

  private handleTouchBegin(e: egret.TouchEvent): void {
      this.touchStartX = e.stageX;
      this.touchStartY = e.stageY; this.TargetStartX = this.x; this.TargetStartY = this.y;
      this.stage.addEventListener(egret.TouchEvent.TOUCH_MOVE, this.handleTouchMove, this);
  }


  private handleTouchEnd(e: egret.TouchEvent): void {
      this.stage.removeEventListener(egret.TouchEvent.TOUCH_MOVE, this.handleTouchMove, this);
  }

  // 人物跟随手指的移动而左右移动
  private handleTouchMove(e: egret.TouchEvent): void {
      const moveX: number = e.stageX - this.touchStartX;
      let moveToX = this.TargetStartX + moveX;

      // 设置左右移动的边界
      if (moveToX > this.stage.stageWidth - this.width / 2) {
          moveToX = this.stage.stageWidth - this.width / 2
      } else if (moveToX < -this.width / 2) {
          moveToX = -this.width / 2;
      }

      // 改变图层的位置
      this.x = moveToX;
  }
```

不过有一点要注意的是当场景卸载时，需要手动用 `removeEventListener` 来移除监听的事件。


##### 2.3.2 飞行物从屏幕飞过

Egret 中自带了 Tween 缓动库，可以用它来帮助我们实现动画。现在我们要完成一个飞行物从屏幕下方飞到上方的动画。

```javascript
this.y = this.stage.stageHeight + this.height; // 保证刚开始的物体在屏幕外

const tw = egret.Tween.get(this);
tw.to({ y: - this.height }, 1200).call(() => console.log('fly out'));  // 从下面飞到上面
```

##### 2.3.3 碰撞检测

我们将玩家的角色和飞行的怪物都看成是矩形，采用 AABB 碰撞来判断两个矩形是否碰撞。

![AABB](https://learnopengl-cn.github.io/img/06/Breakout/05/02/collisions_overlap.png)

AABB 用代码实现也是非常的简单：
```javascript
collisionsDetection(one: egret.DisplayObject, two: egret.DisplayObject): boolean {
    // x 轴碰撞
    let collisionX: boolean = one.x + one.width >= two.x &&
        two.x + two.width >= one.x;
    // y 轴碰撞
    let collisionY: boolean = one.y + one.height >= two.y &&
        two.y + two.height >= one.y;
    // 只有两个轴向都有碰撞时才碰撞
    return collisionX && collisionY;
},
```

#### 2.4 完成游戏

将刚才游戏场景添加至舞台上，咱们整个游戏就完成了。完成后的代码在 [这里](https://github.com/pspgbhu/egret-sample-game)

## 性能优化的一些最佳实践

在游戏领域，性能优化一项是一个重中之重的问题，还好 Egret 官方整理出了一些关于游戏优化方面的最佳实践，建议大家有空还是需要多看看。[Egret 深入了解性能优化](http://edn.egret.com/cn/docs/page/287)


## 对比与传统 DOM 游戏开发

**优势项：**

1. 数倍于 DOM 游戏性能。
2. 更高的游戏开发效率。
3. 不需要过多考虑兼容性，可以将心思完全放在开发上。
4. 更强大的能力，更多的可能性，能满足更多的复杂的场景。

**劣势项：**

1. 对于 WEB 前端开发工程师来说，用 canvas 游戏引擎需要一定的学习成本。

**总结：**

总而言之，如果想做一款出色的 HTML5 游戏的话，无论游戏大小，都是推荐使用游戏引擎制作的


## Reference
- [HTML5游戏引擎深度测评](http://www.jianshu.com/p/0469cd7b1711)
- [Egret 深入了解性能优化](http://edn.egret.com/cn/docs/page/287)
- [显示优化 - addChild 和 removeChild](https://github.com/xinfangke/egretDocsTest/blob/master/test5.ts)
- [OpenGL 碰撞检测](https://learnopengl-cn.github.io/06%20In%20Practice/2D-Game/05%20Collisions/02%20Collision%20detection/)
