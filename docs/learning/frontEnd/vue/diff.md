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

标记好节点位置后，就进入到 while 循环中，这是 Diff 算法的核心流程，分情况进行新老节点的比较并移动对应的 VNode 节点。while 循环的退出条件是**直到老节点或者新节点的开始位置大于结束位置**。

```javascript
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    ....//处理逻辑
}
```

循环过程中首先对新老 VNode 节点的头尾进行比较，寻找相同节点，如果有相同节点满足`sameVnode`(可以复用的相同节点)则直接进行`patchVnode`(该方法进行节点复用处理)，并且根据具体情形，移动新老节点的 VNode 索引，以便进入下一次循环处理，一共有 5 种情形：

1. 当新旧 VNode 节点的 start 满足 sameVnode 时，直接 patchVnode,同时新旧 VNode 节点的开始索引加 1

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

2. 当新旧 VNode 节点的 end 满足 sameVnode 时，也是直接 patchVnode，同时新旧 VNode 节点的结束索引减 1

```javascript
else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      }
```

3. 当旧 VNode 节点的 start 和新 VNode 节点的 end 满足 sameVnode 时，说明这次数据更新好 oldStartVnode 已经跑到 oldEndVnode 后面了。这时候 patchVnode 后，还需要将当前真是 DOM 节点移动到 oldEndVnode 的后面，同事旧的 VNode 节点开始索引加 1，新 VNode 节点的结束索引减 1

```javascript
else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      }
```

4. 当旧的 VNode 节点的 end 和新 VNode 节点的 start 满足 sameVNode 的时候，说明这次数据更新后 oldEndVnode 跑到 oldStartVNode 的前面了。这时候 patchVNode 后，还需要将当前真是 DOM 节点移动到 oldStartVNode 的前面，同时旧的 VNode 节点结束索引减 1，新的 VNode 节点的开始索引加 1

```javascript
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      }
```

5. **如果都不满足以上四种情形，说明没有相同的节点可以复用**。于是则通过查找事先建立好的以旧的 VNode 为 key 值，对应 index 序列为 value 值的哈希表。从这哈希表中找到与 newStartVNode 一致 key 的旧的 VNode 节点，如果两者满足 sameVNode 的条件，在进行 patchVNode 的同事会将这个真实 DOM 移动到 oldStartVNode 对应的真实 DOM 的前面；如果没有找到，则说明当前索引下的新的 VNode 节点在旧的队列中不存在，无法复用就直接调用 createELm 创建一个新的 DOM 节点放到当前 newStartIdx 的位置

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

比较结果：oldS 和 newE 相等，需要把节点 a 移动到 newE 所对应的位置，就是末尾。同时 oldS++,newE--
![diff7](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff7.png)

- 第二步：

```js
(oldS = b), (oldE = c);
(newS = b), (newE = e);
```

比较结果：oldS 和 newS 相等，需要把 b 节点移动到 newS 对应的位置，oldS++,newS++
![diff8](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff8.png)

- 第三步：

```js
(oldS = c), (oldE = c);
(newS = c), (newE = e);
```

比较结果 ：oldS、oldE 和 newS 相等，需要把节点 c 移动到 newS 对应的位置，oldS++,newS++
![diff9](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff9.png)

- 第四步： oldS>oldE,说明 oldCh 已经遍历完了，但是 newCh 还没遍历完，这时就需要将多出来的节点插入到真实 DOM 的对应位置

![diff10](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/diff10.png)

> [15 张图，20 分钟吃透 Diff 算法核心原理，我说的！！！](https://juejin.cn/post/6994959998283907102)

> [Vue 的 diff 算法解析](https://www.infoq.cn/article/udlcpkh4iqb0cr5wgy7f)
