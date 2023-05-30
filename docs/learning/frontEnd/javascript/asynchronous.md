# JavaScript 的异步编程

上一篇讲解 JS 事件执行机制的文章我们了解到 JS 作为一个“单线程”语言为了解决可能产生的阻塞问题，JavaScript 将任务的执行模式分为了“同步”和“异步”。今天我们来学习下 JavaScript 的异步编程。

## 回调函数（callback）

回调函数应该是我们最早接触的异步编程了。那**回调函数**的定义是什么呢？在 JavaScript 中，函数即对象。我们可以把函数作为参数传递给其他函数，在外部函数调用它。就像下面这个 demo

```javascript
function print(callback) {
  callback();
}
```

上面的代码中`print()`函数将另一个函数作为参数，并在函数体中调用它。在 JavaScript 中，我们就叫它“回调”，而被传递给另一个函数作为参数的函数就是**回调函数**。回调函数可以确保函数在某个任务执行完成前都不运行，在任务执行完成后立即执行。当然我们也可以使用**匿名函数**和**箭头函数**来写回调函数。

```javascript
//匿名函数
setTimeout(function () {
  console.log("This message is shown after 3 seconds");
}, 3000);
//箭头函数
setTimeout(() => {
  console.log("This message is shown after 3 seconds");
}, 3000);
```

**回调函数**的优点是书写简单，学习成本低。但是它也存在一个致命的缺点，那就是**回调地狱（callback hell）**。举个例子，假如我们业务中多个 http 请求存在先后依赖，那我们就可能写出下面的代码

```javascript
ajax(url1, () => {
  // 处理逻辑1
  ajax(url2, () => {
    // 处理逻辑2
    ajax(url3, () => {
      // 处理逻辑3
    });
  });
});
```

上面的代码是很难阅读和维护的，各个部分之间高度耦合，此外它不能使用`try catch`捕获错误，无法直接`return`。

## 事件监听

**事件监听**可以实现异步的任务不取决于代码的顺序，而是取决于某个事件是否发生。
我们来实现这样一个需求：有两个函数`f1`和`f2`,`f2`必须在`f1`执行完毕后才能执行。这里我们采用`jQuery`的写法

```javascript
function f1() {
  setTimeout(function () {
    // ...
    f1.trigger("done");
  }, 1000);
}
function f2() {
  console.log("f2");
}
fn1.on("done", f2);
```

上面的 fn1 中的`f1.trigger("done")`表示触发`done`事件，而通过`f1.on("done", fn2)`绑定上的`f2`就会立即被执行。

其实我们可以看出来这也是**回调函数**的一个变种。这种方法的优点是容易理解，每个事件可以绑定多个函数，可以“去耦合”，利于实现**模块化**。缺点则是程序会变成事件驱动型，运行流程不明晰（其实就是阅读代码时会十分跳跃）。

## 发布订阅

我们假定存在一个“消息中心”，某个任务执行完成，就会向这个“消息中心”**发布（publish）**一个消息，而在之前**订阅（subscribe）**过这个消息的任务就会获取到这个消息开始执行。这就叫做**发布/订阅模式（publish-subscribe pattern）**，也叫做**观察者模式**

```javascript
jQuery.subscribe("done", f2); //f2 向消息中心订阅done消息
function f1() {
  setTimeout(function () {
    // ...
    jQuery.publish("done"); //f1 向消息中心发布done消息，此时订阅done消息的f2执行
  }, 1000);
}
jQuery.unsubscribe("done", f2); //f2 执行结束后可以取消done消息订阅
```

这种方法与“事件监听”比较相似，但是这种方法我们可以通过**消息中心**获取到存在多少消息，每个消息有多少订阅者，从而监控程序。

## Promise

Promise 是最早是有社区提出和实现的，ES6 将其写入了语言标准，统一了用法，原生提供了`Promise`对象。

### Promise 的三种状态

- Pending:Promise 对象实例创建时候的初始状态
- Fulfilled:成功状态
- Rejected:失败状态

![asynchronous1](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/asynchronous1.png)

`Promise`对象有以下两个特点：

1.对象的状态不受外界影响。`Promise`对象代表一个异步操作，只有异步操作的结果可以决定当前是哪一种状态，任何其他操作都无法改变这个状态。

2.一旦状态改变，就不会再变，任何时候都可以得到这个结果。`Promise`对象的状态改变只有两种：从`pending`变成`fulfilled`和从`pending`变为`rejected`。只要这两种情况发生，状态就凝固了，这时就称为**resolved(已定型)**。如果改变已发生，你再对`Promise`对象添加回调函数，也会立即得到这个结果。

```javascript
let p = new Promise((resolve, reject) => {
  reject("reject");
  resolve("success"); //无效代码不会执行
});
p.then(
  (value) => {
    console.log(value);
  },
  (reason) => {
    console.log(reason); //reject
  }
);
```

当我们在构造`Promise`的时候，构造函数内部的代码是立即执行的

```javascript
let promise = new Promise(function (resolve, reject) {
  console.log("Promise");
  resolve();
});

promise.then(function () {
  console.log("resolved.");
});

console.log("Hi!");
```

上面代码中，Promise 新建后立即执行，所以首先输出的是 Promise。然后，then 方法指定的回调函数，将在当前脚本所有同步任务执行完才会执行，所以 resolved 最后输出。

### Promise 的链式调用

- 每次调用返回的都是一个新的`Promise`实例（这就是`then`可以链式调用的原因）
- 如果`then`中返回的是一个结果，会把这个结果传递下一次`then`中的成功回调
- 如果`then`中出现异常，会走下一个`then`的失败回调
- 在`then`中使用了 return，那么 return 的值会被`Promise.resolve()`包装
- `then`中可以不传递参数，如果不传递会穿透到下一个`then`中
- `catch`会捕获到到没有被`try catch`的异常

我们看几个 demo 来帮助理解一下上面的内容：

```javascript
Promise.resolve(1)
  .then((res) => {
    console.log(res);
    return 2; //包装成Promise.resolve(2)
  })
  .catch((err) => 3)
  .then((res) => console.log(res));
```

上面的例子会输出`1 2`,第一个`console.log(res)`,此时`res`是 resolve 内传递的`1`;然后`return 2`会包装成`Promise.resolve(2)`;`catch((err) => 3)`虽然有结果`return 3`,但因为没有错误抛出所以不会执行；最后`then((res) => console.log(res))`还是输出第二次`return 2`的结果

```javascript
// 例2
Promise.resolve(1)
  .then((x) => x + 1)
  .then((x) => {
    console.log(x);
    throw new Error("My Error");
  })
  .catch(() => 1)
  .then((x) => {
    console.log(x);
    x + 2;
  })
  .then((x) => console.log(x))
  .catch((err) => console.log(err));
```

上面的例子会输出`2 1 undefined`,第一个`console.log(x)`,初始的`resolve(1)`经过第一次`then`方法的加 1 操作变成了 2;随后抛出错误,被`catch(() => 1)`捕获返回数字 1；随后`then`方法内输出`1`;然后执行下一个`then`因为上一个`then`方法没有 return 内容，所以输出`undefined`,此时没有未被捕获的异常最后的`catch`不执行

```javascript
// 例3
Promise.resolve(1)
  .then(function (data) {
    throw new Error("Whoops!"); //then中出现异常,会走下一个then的失败回调
  }) //由于下一个then没有失败回调，就会继续往下找，如果都没有，就会被catch捕获到
  .then(function (data) {
    console.log("data");
  })
  .then(null, function (err) {
    console.log("then", err.message);
  })
  .catch(function (err) {
    console.log("error");
  });
```

上面的例子会输出`"then","Whoops!"`,在第一个`then`方法中抛出的异常会走下一个`then`的失败回调，由于下一个 then 没有失败回调，就会继续往下找；在第三个`then`方法中由于它定义了第二个参数（**如果是 promise 内部报错，reject 抛出错误后，then 的第二个参数和 catch 方法都存在的情况下，只有 then 的第二个参数能捕获到，如果 then 的第二个参数不存在，则 catch 方法会捕获到**），所以被第二个参数捕获到异常执行回调内容；最后的`catch`就不会执行

`Promise`的优点是可以将异步操作以同步操作的流程表达出来，避免了层层嵌套的回调函数。此外,`Promise`对象提供统一的接口，使得控制异步操作更加容易。
`Promise`也有一些缺点。首先，无法取消`Promise`，一旦新建它就会立即执行，无法中途取消。其次，如果不设置回调函数，`Promise`内部抛出的错误，不会反应到外部。第三，当处于`pending`状态时，无法得知目前进展到哪一个阶段（刚刚开始还是即将完成）。

## 生成器 Generators/ yield

Generators 函数是 ES6 提供的一种异步编程解决方案，语法行为与传统函数完全不同。Generator 最大的特点就是可以控制函数的执行。我们可以把 generators 理解成一段**可以暂停并重新开始执行的函数**。`function*`是定义 generators 函数的关键字，`yield`是一个操作符，generators 可以通过 yield 暂停自己执行。另外，**generators 可以通过 yield 接受输入和对外输出**。我们来看下面的代码：

```javascript
function* foo(x) {
  let y = 2 * (yield x + 1);
  let z = yield y / 3;
  return x + y + z;
}
let it = foo(5);
console.log(it.next()); // => {value: 6, done: false}
console.log(it.next(12)); // => {value: 8, done: false}
console.log(it.next(13)); // => {value: 42, done: true}
```

我们来逐行分析上面的代码：

- 首先 Generator 函数调用会返回一个迭代器
- 当执行第一次`next`时，传参会被忽略，并且函数会暂定在`yield x+1`处，所以返回`5+1=6`
- 当执行第二次`next`时,传入的参数`12`会被当作上一个 yield 表达式的返回值，此时`let y =2*12`,所以第二个 yield 等于`2*12/3=8`
- 当执行第三次`next`时，传入的参数`13`就会被当作上一个 yield 表达式的返回值，所以`z=13 x=5 y =24`相加等于`42`

在现实开发中，原生`generator`的写法并不是十分友好，我们可以通过`co`库优化代码。`co` **是一个为 Node.js 和浏览器打造的基于生成器的流程控制工具，借助于 Promise，你可以使用更加优雅的方式编写非阻塞代码**。我们可以通过下面的命令安装`co`:

```
npm install co
```

通过调用`co()`,我们可以得到一个`Promise`对象；我们来下下面这个例子：有三个本地文件，分别 1.txt,2.txt 和 3.txt，内容都只有一句话，下一个请求依赖上一个请求的结果，想通过 Generator 函数依次调用三个文件

```
//1.txt文件
2.txt
```

```
//2.txt文件
3.txt
```

```
//3.txt文件
结束
```

```
let fs = require('fs')
function read(file) {
  return new Promise(function(resolve, reject) {
    fs.readFile(file, 'utf8', function(err, data) {
      if (err) reject(err)
      resolve(data)
    })
  })
}

function* r() {
  let r1 = yield read('./1.txt')
  let r2 = yield read(r1)
  let r3 = yield read(r2)
  console.log(r1)
  console.log(r2)
  console.log(r3)
}
let co = require('co')
co(r()).then(function(data) {
  console.log(data)
})
// 2.txt=>3.txt=>结束=>undefined
```

## async/await

使用`async/await`,我们可以简单的达到上面生成器所坐到的工作，它有如下特点：

- `async/await`是基于`Promise`实现的，它不是`Promise`的语法糖，不能用于普通的回调函数；
- `async/await`是非阻塞的
- `async/await`使得异步代码看起来像同步代码
  **我们如果给一个函数加上 async,那么该函数会返回一个 Promise**

```javascript
async function async1() {
  return "1";
}
console.log(async1()); // -> Promise {<resolved>: "1"}
```

```javascript
let fs = require("fs");
function read(file) {
  return new Promise(function (resolve, reject) {
    fs.readFile(file, "utf8", function (err, data) {
      if (err) reject(err);
      resolve(data);
    });
  });
}
function readAll() {
  read1();
  read2(); //这个函数同步执行
}
async function read1() {
  let r = await read("1.txt", "utf8");
  console.log(r);
}
async function read2() {
  let r = await read("2.txt", "utf8");
  console.log(r);
}
readAll(); // 2.txt 3.txt
```

## 总结

1.  `JS` 异步编程的进化史：`callback>>promise>>generator>>async/await`
2.  `async/await` 函数的实现，就是将 `generator` 函数和自动执行器，包装在一个函数里面
3.  `async/await` 函数相对于 `Promise` 的优势如下：

    1. 处理`then`的调用链，能更加清晰准确的书写代码
    2. 可以优雅的处理回调地狱
    3. 在使用`await`时要注意，当多个异步操作没有**依赖关系**时不要使用`await`,它会导致性能问题；我们可以使用`Promise.all`来代替

4.  `async/await`函数对 `Generator` 函数的改进如下：

    1. 内置执行器，`async`函数的执行如普通函数相同
    2. 更广的适用性，`await`命令后面可以跟`Promise`对象，也可以是原始类型的值（数字、字符串和布尔值，但此时等同于同步操作）
    3. 更好的语义性，`async` 表示函数里有异步操作，`await` 表示紧跟在后面的表达式需要等待结果。

    <!-- ## 参考文献

> [JS 异步编程六种方案](https://juejin.cn/post/6844903760280420366#heading-2)

> [什么是回调函数？](https://www.freecodecamp.org/chinese/news/javascript-callback-functions/)

> [ECMAScript 6 入门 Promise 对象](https://es6.ruanyifeng.com/#docs/promise) > [Generators 深度解读](https://juejin.cn/post/6844903517979688967)
> -->
