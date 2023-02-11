---
title: Vue中的Diff算法
category: Vue原理
cover: http://lc-llaarrzl.cn-n1.lcfile.com/I7MmFRiwQC9TwhaHGYtdL9pxDGHfHib2/1.png
---

# 前置知识：虚拟 DOM

虚拟 DOM 节点是一个 JS 对象，用这个 JS 对象可以表示 DOM 节点、组件节点等，创建一个虚拟 DOM 节点比创建一个 DOM 节点的代价要小很多。有了虚拟 DOM，能提高我们的研发体验和效率，同时也能解决跨平台的问题

在 Vue 中，通常用 VNode 指代一个虚拟 DOM 节点，我们最终会根据 VNode 生成 DOM 节点. 在 React 中，通过 React.createElement 也能生成一个虚拟 DOM 节点（ReactElement）

# Vue2中的双端Diff算法

Vue2中的双端Diff是旧的一组VNode( 旧子节点 )和新的一组VNode( 新子节点 )进行对比

## 第一轮遍历

所谓的双端, 表示在新旧子节点的数组中, 各用两个指针指向头尾节点, 在遍历过程中, 头尾指针不断靠拢. 因此, 用newStartIndex和newEndIndex分别指向新子节点中未处理的头尾节点, 用oldStartIndex和oldEndIndex分别指向旧子节点中未处理节点的头尾节点

现在, 我们用「新前」表示新子节点中未处理节点的第一个节点; 用「新后」表示新子节点中未处理的最后一个节点; 「旧前」表示旧子节点中未处理节点的第一个节点; 用「旧后」表示旧子节点中未处理节点的最后一个节点

每遍历到一个节点, 就尝试进行双端比较: 「新前 vs 旧前」、「新后 vs 旧后」、「新后 vs 旧前」、「新前 vs 旧后」, 如果匹配成功, 更新双端的指针 . 比如, 新旧子节点通过「新前 vs 旧后」匹配成功，那么 newStartIndex += 1，oldEndIndex -= 1

如果新旧子节点通过「新后 vs 旧前」匹配成功, 还需要将「旧前」对应的DOM节点插入到「旧后」对应的DOM节点之后. 如果新旧节点通过「新前 vs 旧后」, 还需要将「旧后」对应的DOM节点插入到「旧前」对应的DOM节点之前

如果通过双端比较都没法找到匹配的节点, 就需要一个React中的existingChildren Map的Map对象了, 在Vue2的diff中, 这个名字叫做oldKeyToIdx Map. 通过这个Map, 遍历时可以尝试根据新子节点的key去找oldIndex, 查找结果会有两种:

找到oldIndex, 即新旧子节点有相同key的节点

如果VNode的type是相同的, 将旧子节点对应的DOM节点插入到「旧前」对应的DOM节点之前

如果VNode的type是不同的, 创建一个新的DOM节点, 并插入到「旧前」对应的DOM节点之前

没找到oldIndex, 需要根据新子节点(VNode)创建DOM元素, 并插入到「旧前」对应的DOM节点之前

简单来说, 第一轮遍历会先尝试比较新旧子节点的双端节点, 如果匹配不成功, 再尝试在旧子节点中找到对应的节点. 至于DOM节点的移动, 需要记住只能移动到「旧前」之前或「旧后」之后 . 如果更新后节点位置被调到前面了 移动时就需要移动到「旧前」之前; 如果更新后节点位置被调到后面了, 移动时就需要移到「旧后」之后

## 第二轮遍历

如果第一轮遍历后, 只剩下新子节点( newStartIndex>newEndIndex ) , 则根据剩余的新子节点(VNode)创建DOM节点, 并依次插入到父级DOM节点最后

如果第一轮遍历后, 只剩下旧子节点(oldStartIndex>oldEndIndex), 则将剩余旧子节点对应的DOM节点依次从父级DOM节点中删除

需要注意的是, Vue在diff过程中, 会直接进行节点的更新/新建/删除操作, 这点和React是不同的

下面展示了Vue2双端Diff的一些例子, 能够更好的消化理解算法:

![UML 图 (8).jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c45e7a6ad91b49f0ac9ec35e15484359~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

# Vue3中的快速Diff算法

注意: Vue3多节点的diff是旧的一组VNode( 旧子节点 ) 和新的一组VNode( 新子节点 )进行对比

## 第一轮遍历

先从新旧子节点的头部节点开始, 一个一个进行对比, 直到遇到不同的节点, 再从新旧子节点的尾部节点开始, 一个一个进行对比, 直到遇到不相同的节点

![UML 图 (9).jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8985f95f59dd4ef083fc5318a39c7708~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

## 第二轮遍历

如果第一轮遍历结束之后还有新子节点或旧子节点未被处理, 那么会有三种情况:

1. 只剩下新子节点         
2. 只剩下旧子节点          
3. 新旧子节点都有剩下的

## 第一种情况: 只剩下新子节点

对于剩余的新子节点, 依次创建对应的DOM节点, 并插到尾部已处理节点之前

![UML 图 (10).jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af7e3ca002e04e13a5899893753a5d63~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

## 第二种情况: 只剩下旧子节点

对于剩余的旧子节点，依次从父级 DOM 节点中删除对应的 DOM 节点

![UML 图 (11).jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac8c2b3d4cdf4422a722c36cc9c288cf~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

----

## 第三种情况: 新旧子节点都有剩下的

这种情况下有可能有节点需要移动, 假设旧子节点的位置索引序列是递增序列, 使用新子节点在旧子节点中的位置索引组合成一个位置索引序列, 如果这个序列是非递增序列, 如果这个序列是非递增序列, 那么就肯定存在节点移动的情况. 进一步思考, 可以发现新的位置索引序列中的最大递增子序列是不需要移动的, 其余索引对应的节点才需要移动

因此, 需要引入求解最长递增子序列的算法

求解最长递增子序列：**贪心 + 二分查找**

---

现在让我们通过一个例子来理解处理流程吧

旧子节点

```html
<div>
  <div key="a">a</div>
  <div key="b">b</div>
  <div key="c">c</div>
  <div key="d">d</div>
  <div key="f">f</div>
  <div key="e">e</div>
</div>
```

新子节点

```html
<div>
  <div key="a">a</div>
  <div key="c">c</div>
  <div key="d">d</div>
  <div key="b">b</div>
  <div key="g">g</div>
  <div key="e">e</div>
</div>
```

要求解最长递增子序列我们就先需要一个数组 newIndexToOldIndexMap（虽然源码中叫 Map，但它实际上是一个数组），这个数组表示新的一组子节点中的节点在旧的一组子节点中的位置索引。这个数组中的 0 值表示对应位置索引的新子节点在旧的一组子节点中不存在匹配的节点，证明需要挂载新的 DOM 节点。因此，只要新子节点在旧的一组子节点中能找到匹配的节点，在记录位置索引到 newIndexToOldIndexMap 前，都需要将位置索引做加 1 处理。

为了构造这个 newIndexToOldIndexMap 数组，需要遍历未处理的旧子节点，遍历过程主要做以下事情：

1. 填充 newIndexToOldIndexMap 数组

1. 确定是否存在需要移动的节点。这个结果将用于判断是否需要求解最长递增子序列，在没有节点移动的情况下，能降低时间复杂度。判断方法本质上和 React 是类似的实现原理，Vue 3 将访问过的最大 newIndex（旧子节点在新的一组子节点中的位置索引）记录到一个叫 maxNewIndexSoFar 的变量中，如果当前 newIndex < maxNewIndexSoFar，证明存在需要移动的节点

1. 更新那些能找到对应新子节点的旧子节点

1. 卸载那些找不到对应新子节点的旧子节点

![1101-1.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8cc0ca3ac56c4bcc89157b0989e79334~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

遍历完旧子节点后，数组 newIndexToOldIndexMap 为：[2, 3, 1, 0]，尽管最长递增子序列是 [2, 3]，但我们需要知道哪些位置的节点是不需要移动的，所以得到这样一个叫 increasingNewIndexSequence 的数组：[0, 1]，表示第一和第二个节点是不需要移动的

接下来从后往前遍历新子节点（假设 i 为当前遍历节点的位置索引）：

如果 newIndexToOldIndexMap[i] 为 0，需要新建 DOM 节点，插入到右边新子节点（VNode）的 DOM 节点之前

如果 increasingNewIndexSequence 数组里不包含 i，证明这个新子节点不在最长递增子序列中，需要将它对应的 DOM 节点需要插入到右边新子节点的 DOM 节点之前

如果 increasingNewIndexSequence 数组里包含 i，证明当前新子节点对应的 DOM 节点不需要移动

注意，遍历新子节点时不再需要更新节点，因为在遍历旧子节点时已经更新过了

![UML 图 (12).jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5712293272e342dd8e179183d2e17f6c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)





























