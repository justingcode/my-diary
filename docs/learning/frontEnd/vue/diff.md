# 图解 diff 算法

## 前言

在上一篇介绍虚拟 DOM 的文章中，我们讲到现在主流框架两大核心是`Virtual DOM`和`Diff`算法，上篇着重讲了`Virtual DOM`，本篇我们就来讲解一下`Diff`算法。

## 什么是 Diff 算法

我们在上面文章纠正了这样一个错误概念**虚拟 DOM 比真实 DOM 快**，正确的说法应该是：**虚拟 DOM 算法操作真实 DOM,性能高于直接操作真实 DOM，虚拟 DOM 算法=虚拟 DOM+Diff 算法**。那`Diff`算法到底是什么呢？我们先看下面这张图：
![diff1](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff1.png)
总结上图就是：**Diff 算法是一种对比算法**。对比两针是`旧虚拟DOM和新虚拟DOM`,对比出是哪个`虚拟节点`更改了，找出这个`虚拟节点`，并只更新这个虚拟节点所对应的`真实节点`,而不用更新其他数据没发生改变的节点，实现精准地更新真实 DOM。

- 使用虚拟 DOM 算法的损耗计算：总损耗=虚拟 DOM 增删改+（与 Diff 算法效率有关）真实 DOM 差异增删改+（准确的）排版与重绘
- 直接操作真实 DOM 的损耗计算：总损耗=真实 DOM 完全增删改+（大量的）排版与重绘

`Diff`算法是一种通过同层的树节点进行比较的高效算法，避免了对树进行逐层搜索遍历，所以时间复杂度只有`O(n)`。`Diff`算法有两个特点：

- 比较只会在同层级进行，不会跨层级比较
  ![diff2](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff2.png)

- 在 diff 比较过程中，循环从两边向中间收拢
  ![diff3](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff3.png)

## Diff 流程

Diff 算法的总体流程如下图
![diff4](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff4.png)
我们还是通过 vue 源码来解读分析 Diff 算法流程，下面把整个流程拆分成三步：

### 第一步

vue 的虚拟 DOM 渲染真实 DOM 的过程中首先会对新老 VNode 的开始合结束位置进行标记：oldStartIdx、oldEndIdx、newStartIdx、newEndIdx

```javascript
let oldStartIdx = 0; // 旧节点开始下标
let newStartIdx = 0; // 新节点开始下标
let oldEndIdx = oldCh.length - 1; // 旧节点结束下标
let oldStartVnode = oldCh[0]; // 旧节点开始vnode
let oldEndVnode = oldCh[oldEndIdx]; // 旧节点结束vnode
let newEndIdx = newCh.length - 1; // 新节点结束下标
let newStartVnode = newCh[0]; // 新节点开始vnode
let newEndVnode = newCh[newEndIdx]; // 新节点结束vnode
```

经过第一步之后，我们初始的新建 VNode 节点如下图所示
![diff5](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff5.png)

### 第二步

标记好节点位置后，就进入到 `while` 循环中，这是 `Diff `算法的核心流程，分情况进行新老节点的比较并移动对应的 `VNode` 节点。`while` 循环的退出条件是**直到老节点或者新节点的开始位置大于结束位置**。

```javascript
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    ....//处理逻辑
}
```

循环过程中首先对新老 `VNode` 节点的头尾进行比较，寻找相同节点，如果有相同节点满足`sameVnode`(可以复用的相同节点)则直接进行`patchVnode`(该方法进行节点复用处理)，并且根据具体情形，移动新老节点的 `VNode `索引，以便进入下一次循环处理，一共有 4 种情形：

1. 当新旧 `VNode` 节点的 `start` 满足 `sameVnode` 时，直接 `patchVnode`,同时新旧 `VNode` 节点的开始索引加 1

```javascript
if (sameVnode(oldStartVnode, newStartVnode)) {
  patchVnode(
    oldStartVnode,
    newStartVnode,
    insertedVnodeQueue,
    newCh,
    newStartIdx
  );
  oldStartVnode = oldCh[++oldStartIdx];
  newStartVnode = newCh[++newStartIdx];
}
```

2. 当新旧 `VNode` 节点的 `end `满足 `sameVnode` 时，也是直接 `patchVnode`，同时新旧 `VNode` 节点的结束索引减 1

```javascript
else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      }
```

3. 当旧 `VNode `节点的 `start `和新 `VNode` 节点的 `end` 满足 `sameVnode` 时，说明这次数据更新好 `oldStartVnode` 已经跑到 `oldEndVnode` 后面了。这时候 `patchVnode` 后，还需要将当前真实` DOM` 节点移动到 `oldEndVnode` 的后面，同事旧的 `VNode` 节点开始索引加 1，新 `VNode` 节点的结束索引减 1

```javascript
else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      }
```

4. 当旧的 `VNode` 节点的`end`和新 `VNode` 节点的 `start` 满足 `sameVNode` 的时候，说明这次数据更新后 `oldEndVnode` 跑到 `oldStartVNode` 的前面了。这时候 `patchVNode` 后，还需要将当前真是 `DOM` 节点移动到 `oldStartVNode` 的前面，同时旧的 `VNode` 节点结束索引减 1，新的 `VNode` 节点的开始索引加 1

```javascript
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      }
```

5. **如果都不满足以上四种情形，说明没有相同的节点可以复用**。于是则通过查找事先建立好的以旧的 `VNode `为 `key` 值，对应 `index` 序列为 `value `值的哈希表。从这哈希表中找到与 `newStartVNode` 一致 `key` 的旧的`VNode` 节点，如果两者满足 `sameVNode `的条件，在进行 `patchVNode `的同时会将这个真实 `DOM` 移动到 `oldStartVNode` 对应的真实 `DOM` 的前面；如果没有找到，则说明当前索引下的新的 `VNode` 节点在旧的队列中不存在，无法复用就直接调用 `createELm `创建一个新的 `DOM` 节点放到当前 `newStartIdx `的位置

```javascript
else {// 没有找到相同的可以复用的节点，则新建节点处理
        /* 生成一个key与旧VNode的key对应的哈希表（只有第一次进来undefined的时候会生成，也为后面检测重复的key值做铺垫） 比如childre是这样的 [{xx: xx, key: 'key0'}, {xx: xx, key: 'key1'}, {xx: xx, key: 'key2'}] beginIdx = 0 endIdx = 2 结果生成{key0: 0, key1: 1, key2: 2} */
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        /*如果newStartVnode新的VNode节点存在key并且这个key在oldVnode中能找到则返回这个节点的idxInOld（即第几个节点，下标）*/
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          /*newStartVnode没有key或者是该key没有在老节点中找到则创建一个新的节点*/
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          /*获取同key的老节点*/
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            /*如果新VNode与得到的有相同key的节点是同一个VNode则进行patchVnode*/
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            //因为已经patchVnode进去了，所以将这个老节点赋值undefined
            oldCh[idxInOld] = undefined
            /*当有标识位canMove实可以直接插入oldStartVnode对应的真实Dom节点前面*/
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            /*当新的VNode与找到的同样key的VNode不是sameVNode的时候（比如说tag不一样或者是有不一样type的input标签），创建一个新的节点*/
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
```

我们来举一个例子，通过图示的方法分析一遍。新旧两个子节点集合以及其首尾指针如下图：
![diff6](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff6.png)

- 第一步：

```js
(oldS = a), (olaE = C);
(newS = b), (newE = A);
```

比较结果：`oldS` 和 `newE` 相等，需要把节点 `a` 移动到 `newE` 所对应的位置，就是末尾。同时 `oldS++`,`newE--`
![diff7](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff7.png)

- 第二步：

```js
(oldS = b), (oldE = c);
(newS = b), (newE = e);
```

比较结果：`oldS` 和 `newS` 相等，需要把` b` 节点移动到 `newS` 对应的位置，`oldS++`,`newS++`
![diff8](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff8.png)

- 第三步：

```js
(oldS = c), (oldE = c);
(newS = c), (newE = e);
```

比较结果 ：`oldS`、`oldE` 和 `newS` 相等，需要把节点` c` 移动到 `newS` 对应的位置，`oldS++`,`newS++`
![diff9](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff9.png)

- 第四步： `oldS>oldE`,说明 `oldCh` 已经遍历完了，但是 `newCh` 还没遍历完，这时就需要将多出来的节点插入到真实 `DOM` 的对应位置

![diff10](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff10.png)
最后附上`updateChildren`的完整代码

```js
function updateChildren(parentElm, oldCh, newCh) {
  let oldStartIdx = 0,
    newStartIdx = 0;
  let oldEndIdx = oldCh.length - 1;
  let oldStartVnode = oldCh[0];
  let oldEndVnode = oldCh[oldEndIdx];
  let newEndIdx = newCh.length - 1;
  let newStartVnode = newCh[0];
  let newEndVnode = newCh[newEndIdx];
  let oldKeyToIdx;
  let idxInOld;
  let elmToMove;
  let before;
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (oldStartVnode == null) {
      oldStartVnode = oldCh[++oldStartIdx];
    } else if (oldEndVnode == null) {
      oldEndVnode = oldCh[--oldEndIdx];
    } else if (newStartVnode == null) {
      newStartVnode = newCh[++newStartIdx];
    } else if (newEndVnode == null) {
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      patchVnode(oldStartVnode, newStartVnode);
      oldStartVnode = oldCh[++oldStartIdx];
      newStartVnode = newCh[++newStartIdx];
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      patchVnode(oldEndVnode, newEndVnode);
      oldEndVnode = oldCh[--oldEndIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldStartVnode, newEndVnode)) {
      patchVnode(oldStartVnode, newEndVnode);
      api.insertBefore(
        parentElm,
        oldStartVnode.el,
        api.nextSibling(oldEndVnode.el)
      );
      oldStartVnode = oldCh[++oldStartIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldEndVnode, newStartVnode)) {
      patchVnode(oldEndVnode, newStartVnode);
      api.insertBefore(parentElm, oldEndVnode.el, oldStartVnode.el);
      oldEndVnode = oldCh[--oldEndIdx];
      newStartVnode = newCh[++newStartIdx];
    } else {
      // 使用key时的比较
      if (oldKeyToIdx === undefined) {
        oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx); // 有key生成index表
      }
      idxInOld = oldKeyToIdx[newStartVnode.key];
      if (!idxInOld) {
        api.insertBefore(
          parentElm,
          createEle(newStartVnode).el,
          oldStartVnode.el
        );
        newStartVnode = newCh[++newStartIdx];
      } else {
        elmToMove = oldCh[idxInOld];
        if (elmToMove.sel !== newStartVnode.sel) {
          api.insertBefore(
            parentElm,
            createEle(newStartVnode).el,
            oldStartVnode.el
          );
        } else {
          patchVnode(elmToMove, newStartVnode);
          oldCh[idxInOld] = null;
          api.insertBefore(parentElm, elmToMove.el, oldStartVnode.el);
        }
        newStartVnode = newCh[++newStartIdx];
      }
    }
  }
  if (oldStartIdx > oldEndIdx) {
    before = newCh[newEndIdx + 1] == null ? null : newCh[newEndIdx + 1].el;
    addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx);
  } else if (newStartIdx > newEndIdx) {
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
  }
}
```

## Diff 算法中的 key

`key` 在 `Diff` 算法中是 `VNode` 的唯一标识，`diff` 算法中的双端两两比较一共有 4 中情况，如果 4 中情况都没有匹配，当设置了 `key` 就会用 `key` 再进行比较。通过这个 `key` 我们可以更准确、快速的复用节点更新视图。复用节点就是需要通过移动元素的位置来进行更新。
能够移动元素的关键在于：我们需要在新旧 `children` 的节点中保存映射关系（`idsInOld` 存在），以便我们能够在旧节点中找到可复用的节点。这时候我们就需要给节点添加唯一标识也就是 `key`。如果没有 `key`,我们是没有办法知道新节点是否被复用的可能性，即没法知道是否可以通过移动预算位置来实现更新。如果知道了映射关系，我们就可以判断新节点是否可复用，只需要遍历新 `children` 的每个节点，并去旧的 `oldKeyToIdx` 哈希表中寻找相同 `key` 值的节点。
如果没有 `key` 或者使用 `index` 作为 `key`，`vue` 会使用一种最大限度减少动态元素并且通过创建节点进行原地复用或就低修改通类型的算法来更新。而使用 `key` 时并进一步通过 `sameVnode` 爬到是否是同一节点。如果是同一节点，它会基于 `key` 的变化来重新排列元素顺序，并且会移除 `key` 不存在的元素。否则，虽然有相同的 `key`（例如 index 作为 `key` 相同）但是是不同的元素，则通过 `createElm` 创建新节点就地修改。
总结来说 `key `的作用如下：

- 如果遇到**完整地触发组件的生命周期钩子或者触发过度**的场景，可以用于强制替换元素、组件而不是重复使用它
- 更准确，基于 `key` 的变化重新排列元素顺序，并且会移除 `key` 不存在的元素，通过移动节点位置，避免原地复用或者就地修改来复用节点，更新视图。有相同父元素的子元素的 `key` 必须唯一，重复的` key` 会渲染错误。
- 更快速，利用 key 生成的 map 对象来获取对应节点，比遍历更快
- 如果不用 `key`（或者用 `index` 作 `key`），会采用就地复用原则：最小化的 `element` 移动，会最大程度在原位置对相同类型的 `element` 做 `patch` 或者复用
- 如果使用 `key`，`vue` 会根据 `key` 的顺序记录 `element`，曾经有 `key` 的 `element` 如果不再出现就直接 `remove` 或者 `destroy`。用 `new Date()`生成的时间戳作为 `key` 会手动触发重新渲染
- 当拥有新值的 `rerender` 作为 `key`，拥有了新 `key` 的组件出现，那么旧 `key` 的组件会被移除，新 key 组件触发渲染

### 为什么不要用 index 作为 key

我们来看下面这个例子：

```javascript
    <div id="app">
      <ul>
        <item
          :key="index"
          v-for="(num, index) in nums"
          :num="num"
          :class="`item${num}`"
        ></item>
      </ul>
      <button @click="change">改变</button>
    </div>
    <script src="./vue.js"></script>
    <script>
      var vm = new Vue({
        name: "parent",
        el: "#app",
        data: {
          nums: [1, 2, 3]
        },
        methods: {
          change() {
            this.nums.reverse();
          }
        },
        components: {
          item: {
            props: ["num"],
            template: `
                    <div>
                       {{num}}
                    </div>
                `,
            name: "child"
          }
        }
      });
    </script>

```

在首次渲染的时候，虚拟 `DOM` 大致如下：

```js
[
  {
    tag: "item",
    key: 0,
    props: {
      num: 1,
    },
  },
  {
    tag: "item",
    key: 1,
    props: {
      num: 2,
    },
  },
  {
    tag: "item",
    key: 2,
    props: {
      num: 3,
    },
  },
];
```

当我们点击按钮将列表进行 `reverse`，那么上面的虚拟 `DOM` 就是 `oldChildren`,而 `newChildren` 如下：

```js
[
  {
    tag: "item",
    key: 0,
    props: {
      num: 3,
    },
  },
  {
    tag: "item",
    key: 1,
    props: {
      num: 2,
    },
  },
  {
    tag: "item",
    key: 2,
    props: {
      num: 1,
    },
  },
];
```

按照我们理解的最合理的操作逻辑，`oldChildren` 的第一个 `vnode` 应该直接完全复用 `newChildren` 的第三个 `vnode`，因为他们是同一个 `vnode`，所有属性都是相同的。
但是在进行 `diff` 过程中，会在**旧首节点和新首节点**用 `sameNode` 对比。这一步命中逻辑，将 `oldChildren` 的第一个 `vnode` 和 `newChildren` 的第一个 `vnode` 进行 `patchVnode`,这时可能会触发一系列视图重新渲染。
如果我们把 `demo` 中的第一个 `li` 删除，按合理的逻辑应该是直接移除 `oldChildren` 第一个子节点，而如果使用 `index` 作为 `key，则是将 `oldChildren` 的最后一个节点删除，并且更新前两个节点的属性

### 为什么不要用随机数作为 key

我们再来看下面这个例子：

```js
<item
  :key="Math.random()"
  v-for="(num, index) in nums"
  :num="num"
  :class="`item${num}`"
/>
```

首先我们的 `oldChildren` 如下：

```js
[
  {
    tag: "item",
    key: 0.6330715699108844,
    props: {
      num: 1,
    },
  },
  {
    tag: "item",
    key: 0.25104533240710514,
    props: {
      num: 2,
    },
  },
  {
    tag: "item",
    key: 0.4114769152411637,
    props: {
      num: 3,
    },
  },
];
```

更新后的 `newChildren` 如下：

```js
[
  {
    tag: "item",
    key: 0.11046018699748683,
    props: {
      num: 3,
    },
  },
  {
    tag: "item",
    key: 0.8549799545696619,
    props: {
      num: 2,
    },
  },
  {
    tag: "item",
    key: 0.18674467938937478,
    props: {
      num: 1,
    },
  },
];
```

上面说到，`diff` 子节点如果没有命中，就会进入 `key` 的对比过程，就是利用旧节点的 `key->index` 的关系建立一个 `map` 映射表，然后用新节点的 `key` 去匹配，如果没有找到就 调用 `createElm` 重新建立一个节点，代码如下：

```js
// 建立旧节点的 key -> index 映射表
oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);

// 去映射表里找可以复用的 index
idxInOld = findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);
// 一定是找不到的，因为新节点的 key 是随机生成的。
if (isUndef(idxInOld)) {
  // 完全通过 vnode 新建一个真实的子节点
  createElm();
}
```

显而易见上面的 `demo` 的更新过程就是：`123->在前面重新创建三个子组件->321123->删除销毁后面三个子组件->321`;本来只用通过移动位置可以完成的更新，现在变成需要新建、销毁组件来完成更新。

> [15 张图，20 分钟吃透 Diff 算法核心原理，我说的！！！](https://juejin.cn/post/6994959998283907102)

> [Vue 的 diff 算法解析](https://www.infoq.cn/article/udlcpkh4iqb0cr5wgy7f) >

> [虚拟 DOM 与 Diff 算法](https://jonny-wei.github.io/blog/vue/vue/vue-diff.html)

> [为什么 Vue 中不要用 index 作为 key？（diff 算法详解）](https://juejin.cn/post/6844904113587634184#heading-0)
