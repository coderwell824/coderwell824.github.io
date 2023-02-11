---
title: React中的Diff算法
category: React原理
cover: http://lc-llaarrzl.cn-n1.lcfile.com/u1sYzSrHdS48UR8wbeSIl28F2Ny4GrUx/diff.png
---

# 前置知识：虚拟 DOM

虚拟 DOM 节点是一个 JS 对象，用这个 JS 对象可以表示 DOM 节点、组件节点等，创建一个虚拟 DOM 节点比创建一个 DOM 节点的代价要小很多。有了虚拟 DOM，能提高我们的研发体验和效率，同时也能解决跨平台的问题

在 Vue 中，通常用 VNode 指代一个虚拟 DOM 节点，我们最终会根据 VNode 生成 DOM 节点. 在 React 中，通过 React.createElement 也能生成一个虚拟 DOM 节点（ReactElement）

# React中的DOM架构

在 React15 及以前，采用了递归的方式创建虚拟 DOM，递归过程是不能中断的。如果组件树的层级很深，递归会占用线程很多时间，造成卡顿。React16 将递归的无法中断的更新重构为异步的可中断更新，推出了新的 Fiber 架构

原本的 ReactElement 只有 children，在中断恢复时，无法找到其兄弟节点和父节点，无法从断点处继续完成渲染工作。而 fiber 节点上能访问到父节点、子节点、兄弟节点，所以即使渲染被打断了，也可以恢复查找未处理的节点。因此，React 需要先生成 ReactElement，再生成 fiber，最后才将变更映射到真实 DOM 节点。这一点和 Vue 有很大不同

React 采用了双缓存的技术，在 React 中最多会存在两颗 fiber 树，当前屏幕上显示内容对应的 fiber 树称为 current fiber 树，正在内存中构建的 fiber 树称为 workInProgress fiber 树。当 workInProgress fiber 树构建并渲染到页面上后，应用根节点的 current 指针指向 workInProgress Fiber 树，此时workInProgress Fiber 树就变为 current Fiber 树

假设我们有这样一段代码：

```react
const App = () => {
  const [count, setCount] = React.useState(0)
  return <div onClick={() => setCount(n => n + 1)}>{count}</div>
}

ReactDOM.createRoot(document.getElementById('root')).render(<App />)
```

那么，对应的 fiber 树会经历如下图所示的变化过程：

<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ccc58df27154105b150ee9a7a24ba10~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?" alt="UML 图.jpg " style="zoom:67%;" />

React 的更新会经历两个阶段：render 阶段 和 commit 阶段。render 阶段是可中断的，commit 阶段是不可中断的

render 阶段会生成 fiber 树，所谓的 diff 就会发生在这个阶段。React 通过深度优先遍历来生成 fiber 树，整个过程与递归是类似的，因此生成 fiber 树的过程又可以分为「递」阶段和「归」阶段

commit 阶段主要执行各种 DOM 操作、生命周期钩子、某些 hook 等

因此，diff 阶段不会直接变更 DOM，而是留到 commit 阶段再做变更

Vue 与 React 不同，它通过递归的形式生成整个虚拟 DOM 树，在 diff 的同时会对 DOM 做变更

# React 18中简单diff算法

React 每次更新时，会将新的 ReactElement（即 React.createElement() 的返回值）与旧的 fiber 树作对比，比较出它们的差异后，构建出新的 fiber 树，因此多节点的diff实际上是用 fiber（旧子节点）和 ReactElement 数组（新子节点）进行对比

## 第一轮遍历

React 团队发现，在实际的场景中，更新节点的情况要大于新增和删除节点的情况，因此第一轮遍历会先尝试更新子节点

遍历逻辑如下：

如果新旧子节点的 key 和 type（节点类型，如 div、p、span、函数组件名）都相同，则根据旧 fiber 和 新 ReactElement 的 props 生成新子节点 fiber

如果新旧子节点的 key 相同，但 type 不同，将根据新 ReactElement 生成新 fiber，旧 fiber 将被添加到它的父级 fiber 的 deletions 数组中，后续将被移除

如果新旧子节点的 key 和 type 都不相同，结束遍历

<img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72f401ece31e418f86c1e40206eb9117~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?" alt="UML 图 (1).jpg " style="zoom:67%;" />

## 第二轮遍历

如果第一轮遍历被提前终止了，意味着还有新 ReactElement 或 旧 fiber 还未被遍历。因此会有第二轮遍历去处理以下三种情况：

只剩旧子节点

只剩新子节点

新旧子节点都有剩

下图分别展示了这三种情况：

<img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84d81f81ddf54fe59a989a48356ff8e3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?" alt="UML 图 (2).jpg " style="zoom:67%;" />

### 第一种情况: 只剩旧子节点

只剩下旧子节点的处理方法很简单，只需要将剩余的旧 fiber 放到父 fiber 的 deletions 数组中，这些旧 fiber 对应的 DOM 节点将会在 commit 阶段被移除

![UML 图 (3).jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ed4b64494db48eb953c216c70feeff7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 第二种情况: 只剩新子节点

对于剩余的新子节点，先创建新的 fiber 节点，然后打上 Placement 标记，我们将在遍历 fiber 树的「归」阶段生成这些新 fiber 对应的 DOM 节点

![UML 图 (4).jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d97dac1167024d30bfc5fe8e3ec4d075~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 第三种情况: 新旧子节点都有剩

这种情况下，需要一个快速的方法帮助我们快速找到某个 ReactElement 在上一次渲染时生成的 fiber 节点。因此，我们需要一个 existingChildren Map，这个 Map 保存了旧 fiber 的 key 到 旧 fiber 的映射关系，我们可以通过新的 ReactElement 的 key 快速在这个 Map 中找到对应的旧 fiber，如果能找到，则能复用旧 fiber 以生成新 fiber；如果找不到，证明要生成新的 fiber，并打上一个 Placement 标志，以便于在 commit 阶段插入该 fiber 对应的 DOM 节点。

我们还需要找到哪些节点的位置发生了变化

假设我们有 a、b、c、d 四个节点，它们的位置索引是 0、1、2、3，是一个递增的序列。在更新后，它们顺序发生了变化，变成了 a、c、b、d，那么它们的位置索引变为 0、2、1、3（继续沿用旧的位置索引），不再是一个递增的序列，因为索引 1 移到了 2 的后面（即 b 原本在 c 前面，更新后被移到了 c 后面），破坏了递增的规律。因此，只需要找到那些破坏了索引递增规律的节点，就知道哪些节点的位置发生了变化。那具体要怎么做呢？

其实，旧 fiber 上有 index 属性，index 属性记录了在上一次渲染时该 fiber 所在的位置索引。现在，我们暂且叫这个旧 fiber 上的 index 属性为 oldIndex，把遍历新子节点过程中访问过的最大 oldIndex 叫为 lastPlacedIndex，那么，只要当前新子节点有对应的旧 fiber，且 oldIndex < lastPlacedIndex，就可以认为该新子节点对应的 DOM 节点需要往后移动，并打上一个 Placement 标志，以便于在 commit 阶段识别出这个需要移动 DOM 节点的 fiber

遍历逻辑如下：

遍历未处理的旧子节点，生成 existingChildren Map

从前到后遍历新子节点

如果能在 existingChildren Map 中找到对应的旧 fiber，根据旧 fiber 生成新 fiber；如果不能，生成新 fiber，并打上 Placement 标志

从 existingChildren Map 中删除已处理的节点

如果新子节点有对应的旧 fiber

当 oldIndex < lastPlacedIndex 时，给新 fiber 打上 Placement 标志；否则，令 lastPlacedIndex = newIndex

如果新子节点没有对应的旧 fiber，创建一个新 fiber 并 打上 Placement 标志

遍历 existingChildren Map，将 Map 中所有节点添加到父节点的 deletions 数组中

![UML 图 (5).jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f287eb10bb94c4f94e8c710a7606e7f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

## DOM 变更

DOM 元素类型的 fiber 节点上存有对 DOM 节点的引用，因此在 commit 阶段，深度优先遍历每个新 fiber 节点，对 fiber 节点对应的 DOM 节点做以下变更：

1. 删除 deletions 数组中 fiber 对应的 DOM 节点

1. 如有 Placement 标志，将节点移动到往后第一个没有 Placement 标记的fiber的DOM节点之前

1. 更新节点。以 DOM 节点为例，在生成 fiber 树的「归」阶段，会找出属性的变更集，在 commit 阶段更新属性

![UML 图 (6).jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7044fc48726b49389413bcf801aeeb10~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

## 最坏的情况

在处理节点往前移的情况，React 的 diff 算法表现得就不太好了。以下图为例，节点 a、b、c、d 变为了 d、a、b、c，如果我们手动处理这种位置变化，只需要一步：将 d 节点移动到 a 前面。但 React 实际上的做法有三步：将 a、b、c 三个节点依次插入到 d 节点后面

这是因为遍历完 d 节点后，lastPlacedIndex 变成了 3，再去遍历 a、b、c、d 时，oldIndex 一定小于 lastPlacedIndex 了

![UML 图 (7).jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71569a9e6e8045b585b0b1d5c3f100ea~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

因此，实际编写代码时，应该尽量避免节点往前移动的操作

# React为什么不使用双端diff

简单地总结React不使用双端diff的原因: 由于双端diff需要向前查找节点, 但每个fiber节点上都没有反向指针, 即前一个fiber通过sibling属性指向后一个fiber, 只能从前往后遍历, 而不能反过来

























