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

- macro-task 大概包括：`script`(整体代码), `setTimeout`, `setInterval`, `setImmediate`, `I/O`, `UI rendering`。

- micro-task 大概包括: `process.nextTick`, `Promise`, `Object.observe`(已废弃), `MutationObserver`(html5 新特性)

&ensp;&ensp;`script`(整体代码)的分类这里是有争议的，网上有以下两个主流观点：

### 1.认为宏任务包含所有 script 代码的

> 所以，我个人的理解是：宏任务便是 JavaScript 与宿主环境产生的回调，需要宿主环境配合处理并且会被放入回调队列的任务都是宏任务。
> 作者：Reed
> 链接：https://juejin.cn/post/6844903814508773383
> 来源：掘金

> 这里可能有小伙伴不清楚宿主环境的概念，我简单描述下：
> 宿主环境是作为 js 运行的一个载体，常见的有浏览器、node.js 等。
> 我们进入正题，除了广义的同步任务和异步任务，我们对任务有更精细的定义：
> macro-task(宏任务)：包括整体代码 script，setTimeout，setInterval;
> micro-task(微任务)：Promise，process.nextTick;
> 作者：ssssyoki
> 链接：https://juejin.cn/post/6844903512845860872
> 来源：掘金

### 2.宏任务不包含所有 script 代码

> 我觉得不能这样理解，首先宏任务和微任务的定义，都是异步的 js 语句。console.log 明显是同步语句。一个 js 文件的执行，应该是主执行栈先执行同步语句，遇到异步语句，放入任务队列。之后执行微任务队列，然后从宏任务队列取出头部的宏任务执行，执行过程中会产生新的微任务队列。这样循环执行，直到宏任务和微任务全部执行完。eventloop
> 作者：爱吃橘子
> 链接：https://segmentfault.com/q/1010000023206213?utm_source=tag-newest
> 来源：segmentfault 思否

> 因为代码不是任务，所以 console.log()这句代码也不是任务，更不是宏任务。
> 虽然社区有人总喜欢列举 Promise.then、MutationObserver 是微任务（这里没列举完全），但是没捋清事件循环机制的前提下，死记硬背这些个“微任务”的话，面试官很容易借此挖坑请你跳。
> 所谓任务，浅显来说就是代码块开始执行的入口(确切地说，是函数栈的入口，但是栈的概念较为复杂，不表)。而在 JS 里，除了“script 整体代码块”之外，所有代码块的入口都是“回调函数”，回调函数被注册到事件后不会马上被执行，而是保存在一个神秘的的地方，保存起来待执行的才能算“任务”，然后才有宏/微任务之分。
> “script 整体代码块”的特殊之处，在于它的入口不是回调函数，但是我们可以想象它被装在一个隐形的函数里，作为回调函数被注册到某个事件里（大概是它解析完成之后会触发的一个事件），这时候这个隐形的函数就成为了一个任务。
> 作者：madRain
> 链接：https://segmentfault.com/q/1010000023206213?utm_source=tag-newest
> 来源：segmentfault 思否

&ensp;&ensp;关于以上的争论这里不做延展，个人赞同第一个观点。因为在网上其实对**同步任务**的定义就很模糊，所以我一直认为**同步**就是 js 代码从上到下的执行过程，极端一点讲我们完全可以忽视**同步任务**这个概念。而`script`整体代码块的执行则是一个宏任务是被**JS 引擎线程**所执行的。这里可以拓展一下`Renderer`进程内的主要线程：

## Renderer 进程内的主要线程

- GUI 渲染线程

  - 负责渲染浏览器界面，解析 HTML，CSS，构建 DOM 树和 RenderObject 树，布局和绘制等。
    当 JS 引擎执行时 GUI 线程会被挂起(相当于被冻结了),
    GUI 更新会被保存在一个队列中等到 JS 引擎空闲时立即被执行。

- JS 引擎线程

  - JS 引擎线程就是 JS 内核，负责处理 Javascript 脚本程序(例如 V8 引擎)
  - JS 引擎线程负责解析 Javascript 脚本，运行代码
  - JS 引擎一直等待着任务队列中任务的到来，然后加以处理

    - 浏览器同时只能有一个 JS 引擎线程在运行 JS 程序，所以 js 是单线程运行的
    - 一个 Tab 页(renderer 进程)中无论什么时候都只有一个 JS 线程在运行 JS 程序

  - GUI 渲染线程与 JS 引擎线程是互斥的，js 引擎线程会阻塞 GUI 渲染线程

    - 就是我们常遇到的 JS 执行时间过长，造成页面的渲染不连贯，导致页面渲染加载阻塞(就是加载慢)
    - 例如浏览器渲染的时候遇到`<script>`标签，就会停止 GUI 的渲染，然后 js 引擎线程开始工作，执行里面的 js 代码，等 js 执行完毕，js 引擎线程停止工作，GUI 继续渲染下面的内容。所以如果 js 执行时间太长就会造成页面卡顿的情况

- 事件触发线程

  - 属于浏览器而不是 JS 引擎，用来控制事件循环，并且管理着一个事件队列(task queue)
  - 当 js 执行碰到事件绑定和一些异步操作(如 setTimeOut，也可来自浏览器内核的其他线程，如鼠标点击、AJAX 异步请求等)，会走事件触发线程将对应的事件添加到对应的线程中(比如定时器操作，便把定时器事件添加到定时器线程)，等异步事件有了结果，便把他们的回调操作添加到事件队列，等待 js 引擎线程空闲时来处理。
  - 当对应的事件符合触发条件被触发时，该线程会把事件添加到待处理队列的队尾，等待 JS 引擎的处理
  - 因为 JS 是单线程，所以这些待处理队列中的事件都得排队等待 JS 引擎处理

- 定时触发器线程
  - setInterval 与 setTimeout 所在线程
  - 浏览器定时计数器并不是由 JavaScript 引擎计数的(因为 JavaScript 引擎是单线程的，如果处于阻塞线程状态就会影响记计时的准确)
  - 通过单独线程来计时并触发定时(计时完毕后，添加到事件触发线程的事件队列中，等待 JS 引擎空闲后执行)，这个线程就是定时触发器线程，也叫定时器线程
  - W3C 在 HTML 标准中规定，规定要求 setTimeout 中低于 4ms 的时间间隔算为 4ms
- 异步 http 请求线程

  - 在 XMLHttpRequest 在连接后是通过浏览器新开一个线程请求
  - 将检测到状态变更时，如果设置有回调函数，异步线程就产生状态变更事件，将这个回调再放入事件队列中再由 JavaScript 引擎执行
  - 简单说就是当执行到一个 http 异步请求时，就把异步请求事件添加到异步请求线程，等收到响应(准确来说应该是 http 状态变化)，再把回调函数添加到事件队列，等待 js 引擎线程来执行

  > 作者：isboyjc
  > 链接：https://juejin.cn/post/6844904050543034376
  > 来源：稀土掘金

  &ensp;&ensp;了解完上面的概念，接下来才是我们的八股文重点：

  ## 事件循环（Event Loop）

  &ensp;&ensp;同步任务都在主线程（JS 引擎线程）上执行，会形成一个**执行栈**
  主线程之外，事件触发线程管理一个**任务队列**，只要异步任务有了运行结果，就在**任务队列**中放一个事件回调。当**执行栈**中的同步任务都执行完毕即 JS 引擎线程闲置，就会从**任务队列**中读取所有的异步任务添加到**执行栈**中执行。我们先看一个简单的 demo：

```javascript
let timeout = function () {
  console.log("我是定时器回调");
};
// 同步任务
console.log("我是同步任务1");
// 异步定时任务
setTimeout(timeout, 1000);
// 同步任务
console.log("我是同步任务2");
```

&ensp;&ensp;按照上面的代码，我们来分析一下执行过程：

- 执行`console.log("我是同步任务1")`
- 执行`setTimeout(timeout, 1000)`,将其移交给**定时器线程**，告诉定时器线程 1s 后将`timeout`回调交给**事件触发线程**，在 1s 后**事件触发线程**会将`timeout`加入到所管理的事件队列中去；
- 再执行`console.log("我是同步任务2")`
- 此时**js 引擎线程**已空闲，其会询问**事件触发线程**的事件队列内部是否有需要执行的回调函数，如果有的话就执行回调，如果没有的话就一直轮询。

&ensp;&ensp;以上的流程总结来说就是：**js 引擎线程**只会执行**执行栈**里的事件，当事件执行完毕就会想**任务队列**进行轮询，读取到事件就放入到**执行栈**中执行，这样重复的操作就是**事件循环（Event Loop）**

![eventLoop2](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/eventLoop2.png)

## 宏任务（macrotask）

&ensp;&ensp;`macrotask`也被成为`task`,我们可以将每次执行栈中的执行的代码当作一个**宏任务**（也包括从**任务队列**中读取到的回调放入到执行栈中）。每个**宏任务**都会从头到尾执行完毕，中间不会执行其他操作。

> **宏任务**和**GUI 渲染**会交替执行。

常见的**宏任务**：

- 主代码块
- setTimeout
- setInterval
- setImmediate ()-Node
- requestAnimationFrame ()-浏览器

## 微任务（microtask）

&ensp;&ensp;`microtask`也被成为`jobs`。上面我们说到**宏任务**执行完后会执行**GUI 渲染**，但其实在**宏任务**和**GUI 渲染**之间，还会将**宏任务**执行期间产生的所有**微任务**执行完。
常见的**微任务**：

- process.nextTick ()-Node
- Promise.then()
- catch
- finally
- Object.observe
- MutationObserver

&ensp;&ensp;所以最后得到的执行顺序是这样的：**宏任务>>微任务>>GUI 渲染>>宏任务...**。下面举几个例子来演示一下：

```javascript
document.body.style = "background:black";
document.body.style = "background:red";
document.body.style = "background:blue";
document.body.style = "background:pink";
```

&ensp;&ensp;直接说现象，页面会直接渲染成粉色。上面都是四行代码同属于一个**宏任务**中，并且没有**微任务**，所以**宏任务**之行结束后，直接执行渲染。而渲染机制会对其进行优化合并，所以视觉效果就是页面直接渲染成粉色。

```javascript
document.body.style = "background:blue";
setTimeout(() => {
  document.body.style = "background:black";
}, 200);
```

&ensp;&ensp;页面先渲染成蓝色，然后变成黑色背景。现象很好推测，但是其中的还是值得研究的地方的--为什么中间会渲染一次蓝色呢？因为上面的代码其实是两个宏任务，代码执行到第二行的时候遇到`setTimeout`，**定时触发器线程**会接管回调在 200ms 后将其回调放入**任务队列**。然后**宏任务**执行完成，进行**gui 渲染**，页面渲染成蓝色。然后**JS 执行线程**读取**任务队列**获取到任务放入执行栈，开启一个新的**宏任务**，执行完毕后再次进行渲染，将页面渲染成黑色。

```javascript
document.body.style = "background:blue";
console.log(1);
Promise.resolve().then(() => {
  console.log(2);
  document.body.style = "background:pink";
});
console.log(3);
```

&ensp;&ensp;再看第三段代码，表现如下

- 控制台打印 1
- 控制台打印 3
- 控制台打印 2
- 页面渲染为粉色

&ensp;&ensp;因为上面的代码为一个**宏任务**：背景色设置为蓝色，打印 1，打印 3；一个**微任务**：打印 2，页面背景设置为粉色；这样安装上面说到的执行顺序，最后经过渲染优化合并，页面直接渲染为粉色就不会出现蓝色的现象。

## 宏任务和微任务的 tips

- 浏览器会先执行一个**宏任务**，紧接着执行当前**执行栈**产生的**微任务**，再进行渲染，然后再执行下一个**宏任务**；
- **微任务**和**宏任务\***不在一个任务队列
  - 例如`setTimeout`是一个**宏任务**，它的回调存在**宏任务队列**,`Promise.then()`是一个**微任务**，它的事件回调存在**微任务队列**，二者并不在一个任务队列中；
  - 以 Chrome 为例，有关渲染的是在渲染进程中执行，渲染进程中的任务（DOM 树构建，js 解析等）需要再主线程执行的任务都是在**主线程**中执行，而浏览器维护了一套时间循环机制，主线程上的任务都会放到**消息队列**中执行，**主线程**会循环**消息队列**从头部取出任务执行，如果执行过程中产生其他任务需要主线程执行的，渲染进程中的其他线程会把该任务塞到**消息队列**的尾部，**消息队列**中的任务都是**宏任务**
  - 微任务的产生：当执行到`script`脚本的时候，js 引擎会为全局创建一个**执行上下文**，在该执行上下文中维护了一个**微任务队列**。当遇到**微任务**，就会吧微任务回调放到**微任务队列**中，当所有的 js 代码都执行完比，在退出**执行上下文**之前回去检查改队列，有回调就执行，没有就退出**执行上下文**。这就是我们所说的**每个宏任务都有一个微任务队列**。
  ## 总结
  &ensp;&ensp;最后还是回到开篇的图，我们用语言描述下那种图的流程：
  &ensp;&ensp;首先，整体`script`作为**第一个宏任务**开始执行，此时会把所有的代码分为**同步任务**和**异步任务**。同步任务会直接进入**主线程**执行,异步任务会再分为**宏任务**和**微任务**。宏任务会进入到`Event Table`中，并在里面注册回调函数，每当指定的事件完成时，`Event Table`会将这个函数移动到`Event Queue`中；微任务也会执行相同的操作，但它是维护自己独立的`Event Table`和`Event Queue`。当主线程内的任务执行完毕后，会检查**微任务**的`Event Queue`,如果有任务就全部执行，如果没有就执行下一个宏任务。如此往复循环就是`Event Loop`。

## Promise,async/await 函数

&ensp;&ensp;`new Promise(()=>{}).then()`,前面的`new Promise()`这一部分是一个构造函数，这是一个同步任务，而后面的`.then()`才是一个异步微任务。我们可以看下面这段代码来直观感受一下

```javascript
new Promise((resolve) => {
  console.log(1);
  resolve();
}).then(() => {
  console.log(2);
});
console.log(3);
```

&ensp;&ensp;上面的代码输出`1 2 3`
&ensp;&ensp;我们知道`async/await`就是`Promise`的语法糖，而`Promise`就是微任务的一种，所以使用`await`关键字与`Promise.then`效果相似

```javascript
setTimeout(() => console.log(4));

async function test() {
  console.log(1);
  await Promise.resolve();
  console.log(3);
}

test();

console.log(2);
```

&ensp;&ensp;上面的代码输出`1 2 3 4`。我们可以这样理解，`await`之前的代码，相当于与`new Promise`一级的同步代码，`await`之后的代码相当于`Promise.then`内的异步代码

## 练习习题

```javascript
function test() {
  console.log(1);
  setTimeout(function () {
    // macro1
    console.log(2);
  }, 1000);
}

test();

setTimeout(function () {
  // macro2
  console.log(3);
});

new Promise(function (resolve) {
  console.log(4);
  setTimeout(function () {
    // macro3
    console.log(5);
  }, 100);
  resolve();
}).then(function () {
  //micro1
  setTimeout(function () {
    // macro4
    console.log(6);
  }, 0);
  console.log(7);
});

console.log(8);
```

&ensp;&ensp;上面的代码输出什么呢？我们来逐行分析一下。

- js 按顺序从上往下执行；
- 执行到`test()`,该方法为**同步方法**，直接执行`console.log(1)`,打印 1；
- test 方法里面的`setTimeout`为**宏任务**，回调记作`macro1`放入宏任务队列；
- 又遇到一个`setTimeout`,依然是**宏任务**，回调记作`macro2`放入宏任务队列；
- 然后是`new Promise`,它是一个**同步任务**，直接执行` console.log(4)`,打印 4；
- `new Promise`内部的`setTimeout`作为**宏任务**我们还是把其回调记作`macro3`放入宏任务队列；
- `Promise.then`是**微任务**，记作`micro1`直接放入微任务队列中；
- `console.log(8)`是**同步任务**，直接执行，打印 8；
- 此时主线程任务执行完毕，检查微任务队列；
- 开始执行**微任务**，执行`micro1`,遇到`setTimeout`,记作`macro4`，放入宏任务队列；
- 继续执行`micro1`，执行同步任务`console.log(7)`,打印 7；
- 微任务执行完毕，第一次循环结束
- 检查宏任务队列，里面存有是个定时器宏任务`macro1 macro2 macro3 macro4`,按定时器延迟时间得到执行顺序，即`Event Queue：macro2、macro4、macro3、macro1`,依次拿出放入执行栈末尾执行（**注：浏览器 event loop 的 MacroTask queue,及时宏任务队列在每次循环中只会读取一个任务**）
- 执行`macro2`,打印 3，检查没有微任务，第二次`event loop`结束
- 执行`macro4`,打印 6，检查没有微任务，第三次`event loop`结束
- 执行`macro3`,打印 5，检查没有微任务，第四次`event loop`结束
- 执行`macro1`,打印 2，检查没有微任务，第五次`event loop`结束
- 最终输出结果 `1 4 8 7 3 6 5 2`

## 参考文献

> [「硬核 JS」一次搞懂 JS 运行机制](https://juejin.cn/post/6844904050543034376#heading-27)

> [一次搞懂-JS 事件循环之宏任务和微任务](https://juejin.cn/post/6873424205791100942)

> [进程、线程、同步、异步、任务队列、微任务、宏任务](https://juejin.cn/post/7040729389918584840)

> [【JS】深入理解事件循环,这一篇就够了!(必看)](https://zhuanlan.zhihu.com/p/87684858)

> [console.log 是异步流？感觉自己貌似踩了个坑](https://segmentfault.com/q/1010000011091088)

> [JS 中，script(整体代码)是怎么入栈的？](https://segmentfault.com/q/1010000011020960)

> [javascript 中 script 整体代码属于宏任务怎么理解呢？](https://segmentfault.com/q/1010000023206213)

> [前端基础进阶（十四）：深入核心，详解事件循环机制](https://www.jianshu.com/p/12b9f73c5a4f)
