---
title: Git 分支合并 merge or rebase ?
comments: true
date: 2017-11-18 23:16:32
categories: Git
tags: Git
---

前些天学习了一些关于 Git 的一些知识，主要是关于分支合并的一些知识。现在在这里做一个总结，也方便以后自己查阅。


值得一提的是，Git 的官方文档写的特别好，图文并茂，是学习 Git 的最好教程。
> [Git 官方文档](https://git-scm.com/book/zh/v2)

## 合并分支 merge or rebase

大多数情况下：

* 去合并一个公共分支到 **master** 上，请使用 merge --no-ff；
* 去合并一个私有本地分支到 **任意** 分支上，请使用 rebase。

具体还是需要根据不同和场景和需求来决定是 merge 还是 rebase，再归根结底还是需要了解 merge 和 rebase 的区别

### 1. merge
需要合并两个公共分支，选用 merge --on-ff 时可以在合并的时候形成一次新的提交，因此可以很清晰的追溯两个分支之间的每一次合并。

#### 1.1 merge --ff 和 merge --on-ff

Git 中 merge 默认的设置就是 merge --ff，ff 是 fast froward 的意思。现在来说一下 --ff 和 --no-ff 的区别。

![merge-base][1]
现在从 master 分支上的 C2 新建一支 dev 分支，并且在 dev 分支上提交了新的 commit C3 和 C4。

此时在 master 分支上通过普通的 merge 暨 merge --ff 去合并 dev 分支。
```bash
git checkout master
git merge dev    # merge 会默认为 merge --ff
```
因为 C2 后，master 分支上没有新的提交，所以 master 可以成功的使用 fast forward 模式合并分支。合并后的如下图所示：

![merge-base][2]

而如果使用 merge --no-ff 合并分支：
```bash
git checkout master
git merge --no-ff dev
```
**merge --no-ff 会形成一次新的提交作为合并节点，也便于以后追溯分支合并历史。**

![merge-base][3]

以上内容应该已经很清楚的表达出了 merge --no-ff 的作用了吧。

#### 1.2 merge --ff 失效的情况：
![merge-base][4]

还是从 C2 处分出 dev 分支，在 dev 分支上进行了 C3 C4 两次提交，但是这时另外一个同事在 master 分支上进行了另一次提交 C5，此时 C3 便不是从 master 上最新一次提交分出来的了。因此此时再执行 `git merge dev` 的话，便无法通过 fast-forward 方式合并，只能再形成一次新的合并提交。

### 2. rebase
这一部分 Git 官网介绍的非常仔细 [Git-分支-变基](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%8F%98%E5%9F%BA)。本文就做一个大概的提炼。

现在我们在 master 分支上 C2 处新建 experiment 分支，并提交了 C4，这时又有人在 master 上进行了一次 C3 提交。
![此处输入图片的描述][5]

此时想用 rebase 来合并 master 和 experiment 分支。
```
git checkout experiment
git rebase master
```
![git-rebase][6]
现在回到 master 分支，进行一次快进合并。
```
git checkout master
git merge experiment
```
![此处输入图片的描述][7]
OK，现在 master 和 experiment 都指向最新的一次提交了。

C4 的基底是 C2，现在 C4' 的基底是 C3，这次的 rebase 形成了一次**变基**操作，需要注意的是，变基后的 C4' 和之前的 C4 除了提交的内容相同外，**再没有其他任何联系**，C4 和 C4' 的哈希值也是不同的。

#### 变基的风险
千万不对任何公共分支进行变基操作！

变基的本质是丢弃了一部分提交，同时又生成了一部分新的提交。

想象一下这个场景：
1. 你和你的同事都在 dev 分支上开发，你的同事在 dev 分支上提交了 C2 C3，并推到了远程仓库。
2. 你在本地 dev 分支上拉取了你同事的提交 C2 C3。
3. 然后你又把 dev 分支通过 rebase 变基合并到了 master 上，此时你本地已经没有了 C2 和 C3 ，只有变基过后的 C2' 和 C3'，但是 C2 和 C3 却还存在于远程仓库里。
4. 这时你想要 push，却发现还要再 pull，但是 pull 完之后诡异的事情发生了，你会发现有两个一模一样的提交，此时的提交记录变成了：C2'---C3'---C2---C3，原来是变基时被你抛弃的提交又被你从远程仓库拉下来了，尴了个尬。

为了避免上述种种情况，最好还是不要在公共分支进行 rebase 操作。

**一种挽救的方法**：
如果真的出现了上面的情况，通过 `git pull --rebase` 还是可以挽救尴尬的情景的。pull --rebase 会帮助我们合并 C2' 和 C2，C3' 和 C3。

## 个人实践分享
- 多人协作开发时，若要合并两个公共分支，请使用 merge --no-ff 进行合并
- 当你从公共分支上切出一个新的个人分支来进行 BUG 修复，或新功能开发时，若要合并会公共分支，请使用 rebase 来合并公共分支和你的个人开发分支。（此处只需留下你的开发提交即可，个人分支合并会主分支的提交节点意义并不大）
- rebase 前千万要注意变基的分支还有没有其他远程副本，如果有的话，请注意用 `pull --rebase` 来解决不必要的提交。

  [1]: https://pspgbhu.github.io/assets/img/merge-base.png
  [2]: https://pspgbhu.github.io/assets/img/merge--ff.png
  [3]: https://pspgbhu.github.io/assets/img/merge--no-ff.png
  [4]: https://pspgbhu.github.io/assets/img/merge-other.png
  [5]: https://git-scm.com/book/en/v2/images/basic-rebase-1.png
  [6]: https://git-scm.com/book/en/v2/images/basic-rebase-3.png
  [7]: https://git-scm.com/book/en/v2/images/basic-rebase-4.png
