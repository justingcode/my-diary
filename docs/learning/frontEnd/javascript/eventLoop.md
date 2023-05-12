# JavaScript 的事件执行机制

&ensp;&ensp;暂时还没有什么计划，想到什么写什么。现阶段为了面试，所以决定以经典八股文之一`JavaScript`的事件执行机制作为开篇之作。

## 概述 JS 异步执行的原理

&ensp;&ensp;`JavaScript`的一个最大的特点之一就是**单线程**。至于为什么要设计成单线程呢？网上最多的答案就是：

> JavaScript 的单线程，与它的用途有关。作为浏览器脚本语言，JavaScript 的主要用途是与用户互动，以及操作 DOM。这决定了它只能是单线程，否则会带来很复杂的同步问题。比如，假定 JavaScript 同时有两个线程，一个线程在某个 DOM 节点上添加内容，另一个线程删除了这个节点，这时浏览器应该以哪个线程为准？
>
> 所以，为了避免复杂性，从一诞生，JavaScript 就是单线程，这已经成了这门语言的核心特征，将来也不会改变。

&ensp;&ensp;既然设计如此，我们就不再纠结于此。但是如果只是单纯的单线程肯定是不满足实际需求的，所以`JavaScript`就设计了两种任务：**同步任务**和**异步任务**。而**异步任务**又细分成**宏任务**和**微任务**。他们的关系如下图所示：

![eventLoop1](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/eventLoop1.png)

&ensp;&ensp;接下来我们分别来研究下**同步任务**、**异步任务**、**宏任务**和**微任务**。

## 同步任务

&ensp;&ensp;同步任务的定义很简单:

> 在主线程上排队执行的任务，只有前一个任务执行完毕，才能执行后一个任务；

## 异步任务

&ensp;&ensp;我们在代码中开启异步任务的方式有很多，常见的有`Promise`、`setTimeout`、`process.nextTick `，而异步任务的定义如下：

> 不进入主线程、而进入”任务队列”的任务，当主线程中的任务运行完了，才会从”任务队列”取出异步任务放入主线程执行。

&ensp;&ensp;而异步任务分为`macro-task`（宏任务）与`micro-task`（微任务），在最新标准中，它们被分别称为 task 与 jobs。
&ensp;&ensp;macro-task 大概包括：`script`(整体代码), `setTimeout`, `setInterval`, `setImmediate`, `I/O`, `UI rendering`。

micro-task 大概包括: `process.nextTick`, `Promise`, `Object.observe`(已废弃), `MutationObserver`(html5 新特性)
