# 一次搞懂 Promise(一)

## Promise 的含义

Promise 是异步编程的一种解决方案，比传统的解决方案--回调函数和事件--更合理和强大。ES6 提供了统一用法，原生提供了`Promise`对象。所谓`Promise`简单来说就是一个容器，里面保存着某个未来才会结束的事件（通常是一个异步操作）的结果。从语法上说，`Promise`是一个对象，从他可以获取异步操作的消息。`Promise`提供统一的 API,各种异步操作都可以用同样的方法进行处理。
`Promise`对象有以下两个特点：

1. 对象的状态不受外界影响。`Promise`对象代表一个异步操作，有三种状态：`pengding`(进行中)、`fulfilled`(已成功)和`rejected`(已失败)。只有异步操作的结果可以决定当前是哪一种状态，任何其他操作都无法改变这个状态。这也和`Promise`的名字相呼应--“**承诺**”。
2. 状态一旦改变，就不会再变，任何时候都可以得到这个结果。`Promise`对象的状态改变只有两个可能：从`pending`变为`fulfilled`和从`pending`变为`rejected`。只要这两种情况发生，状态就凝固了不会再改变了，这时就成为`resolved`(已定型)。如果改变已经发生，你再对`Promise`对象添加回调函数，也会立即得到结果。**这与事件（evenet）完全不同，事件的特点是：如果你错过了它再去监听它，是得不到结果的。**

## 基本用法

ES6 规定`Promise`对象是一个构造对象，用来生成`Promise`实例。下面的代码创造了一个`Promise`实例：

```javascript
const promise = new Promise(function(resolve, reject) {
  // ... some code

  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(error);
  }
});
```

`Promise`构造函数接受一个函数作为参数，该函数的两个参数分别是`resolve`和`reject`两个函数。

- `resolve`函数，它的作用是将`Promise`对象的状态从**未完成**变成**成功**（`pending`变成`resolved`）,在异步操作成功时调用，并将异步操作结果作为参数传递出去；
- `rejected`函数，它的作用是将`Promise`对象的状态从**未完成**变成**失败**（`pending`变成`rejected`）,在异步操作失败时调用，将异步操作报出的错误作为参数传递出去。

`Promise`实例生成后，可以用`then`方法分别指定`resolved`状态和`rejected`状态的回调函数

```javascript
promise.then(
  function (value) {
    // success
  },
  function (error) {
    // failure
  }
);
```

`then`方法可以接受两个回调函数作为参数。第一个回调是`Promise`对象的状态变成`resolved`时调用；第二个回调是变为`rejected`时调用。这两个函数都是可选的，它们都接收`Promise`对象传出的值作为参数：

```javascript
function timeout(ms) {
  return new Promise((resolve, reject) => {
    setTimeout(resolve, ms, "done");
  });
}

timeout(100).then((value) => {
  console.log(value);
});
```

上面代码中`timeout`方法返回一个`Promise`实例，表达一段时间后才会发生的结果。过了指定的事件，`Promise`实例变成`resolved`状态，就会触发`then`方法绑定的回调函数。

`Promise`新建之后会立即执行

```javascript
let promise = new Promise(function (resolve, reject) {
  console.log("Promise");
  resolve();
});

promise.then(function () {
  console.log("resolved.");
});

console.log("Hi!");

// Promise
// Hi!
// resolved
```

上面的代码中`Promise`新建之后立即执行，所以首先输出的是`Promise`。然后`then`方法指定的回调会在当前脚本所有同步任务执行完后执行，所以`resolved`最后输出。
如果调用`resolve`函数和`reject`函数带有参数，那么它们的参水会被传递给回调函数。`reject`函数的参数通常是`Error`对象的实例，标识抛出的错误；`resolve`函数的参数除了正常的值以外还可能是一个新的`Promise`实例，如下：

```javascript
const p1 = new Promise(function (resolve, reject) {
  // ...
});

const p2 = new Promise(function (resolve, reject) {
  // ...
  resolve(p1);
});
```

上面的代码中，`p1`和`p2`都是`Promise`的实例，但是`p2`的`resolve`方法将`p1`作为参数，即一个异步操作的结果是返回另一个一部操作。这时`p1`的状态会传递给`p2`.如果`p1`的状态是`pending`,那么`p2`的回调就会等待`p1`的状态改变；如果`p1`的状态已经是`resolved`或者`rejected`,那么`p2`的回调就会立即执行。

```javascript
const p1 = new Promise(function (resolve, reject) {
  setTimeout(() => {
    console.log("p1");
    reject(new Error("fail"));
  }, 3000);
});

const p2 = new Promise(function (resolve, reject) {
  setTimeout(() => {
    () => console.log("p2");
    resolve(p1);
  }, 1000);
});

p2.then((result) => console.log(result)).catch((error) => console.log(error));
//p2
//p1
// Error: fail
```

上面的代码中`p1`是一个`Promise`,3 秒之后变成`rejected`。`p2`的状态在 1 秒后改变，`resolve`方法返回的是`p1`。由于`p2`返回的是另一个`Promise`,导致`p2`自己的状态无效了，由`p1`的状态决定`p2`的状态。所以后面的`then`语句都变成针对`p1`。又过来 2 秒，`p1`变成`rejected`,触发`catch`。

注意：调用`resolve`或`reject`并不会终止`Promise`的参数函数的执行。

```javascript
new Promise((resolve, reject) => {
  resolve(1);
  console.log(2);
}).then((r) => {
  console.log(r);
});
// 2
// 1
```

上面的代码调用`resolve(1)`之后`console.log(2)`还是会执行，并且会先执行，因为微任务总是在本轮宏任务结束后执行。
一般来说，调用 `resolve` 或 `reject` 以后，`Promise` 的使命就完成了，后继操作应该放到 `then` 方法里面，而不应该直接写在 `resolve` 或 `reject` 的后面。所以，最好在它们前面加上 `return` 语句，这样就不会有意外。

```javascript
new Promise((resolve, reject) => {
  return resolve(1);
  // 后面的语句不会执行
  console.log(2);
});
```

## Promise.property.then()

`Promise`实例具有`then`方法，它是定义在原型对象`Promise.property`上的。它的作用是为`Promise`实例添加状态改变时的回调函数。`then`方法的第一个参数是`resolved`状态的回调函数，第二个参数是`rejected`状态的回调函数，它们都是可选的。
`then`方法返回的是一个新的`Promise`实例（不是原来的实例，否则会形成死循环）。因此我们可以使用**链式调用**

```javascript
getJSON("/post/1.json")
  .then((post) => getJSON(post.commentURL))
  .then(
    (comments) => console.log("resolved: ", comments),
    (err) => console.log("rejected: ", err)
  );
```

上面的代码中，第一个`then`返回的是一个新的`Promise`对象，第二个`then`就会等待这个新的`Promise`对象状态发生改变如果变为`resolved`就调用第一个回调，如果变为`rejected`就调用第二个回调。

## Promise.property.catch()

`Promise.prototype.catch()`方法是`.then(null, rejection)`或`.then(undefined, rejection)`的别名，用于指定发生错误时的回调函数。如果`Promise`状态已经变成`resolved`,在抛出错误是无效的。`Promise`对象的错误具有“冒泡”性质会一直向后传递，直到被捕获位置。也就是说错误总是就被下一个`catch`语句捕获。

```javascript
getJSON("/post/1.json")
  .then(function (post) {
    return getJSON(post.commentURL);
  })
  .then(function (comments) {
    // some code
  })
  .catch(function (error) {
    // 处理前面三个Promise产生的错误
  });
```

上面的代码一共有三个`Promise`,他们之中任务一个抛出错误都会被最后的`catch`所捕获。**一般来说不建议使用`then`的第二个参数，要保持总是使用`catch`方法。**
跟传统的`try/catch`不同，如果没有使用`catch`方法指定错误处理的回调，`Promise`对象抛出的错误不会传递到外层代码。也就是说`Promise`内部的错误不会影响到外部代码即**Promise 会吃掉错误**。

```javascript
const someAsyncThing = function () {
  return new Promise(function (resolve, reject) {
    // 下面一行会报错，因为x没有声明
    resolve(x + 2);
  });
};

someAsyncThing().then(function () {
  console.log("everything is great");
});

setTimeout(() => {
  console.log(123);
}, 2000);
// Uncaught (in promise) ReferenceError: x is not defined
// 123
```

上面的代码错误产生在`Promise`实例创建内部，浏览器执行到这一行会打印出错误但是并不会终止脚本执行，2 秒后还是会打印出`123`。
再看下面的例子：

```javascript
const promise = new Promise(function (resolve, reject) {
  resolve("ok");
  setTimeout(function () {
    throw new Error("test");
  }, 0);
});
promise.then(function (value) {
  console.log(value);
});
// ok
// Uncaught Error: test
```

上面的代码`Promise`指定再下一轮事件循环中再抛出错误。到了那时候`Promise`的状态已经改变运行已经结束，所以这个错误是在`Promise`函数体外抛出的，会冒泡到最外层成为**未捕获的错误**。`cacth`方法也会返回新的`Promise`,如果没有报错会直接调过`catch`往下继续执行。`catch`方法中也可以再次抛出错误，抛出的错误会被后续的`catch`捕获。

## Promise.prototype.finally()

`finally()`方法用于指定不管`Promise`对象最后状态如何，都会执行的操作。`finally`方法不接受任何参数，这意味着没有办法知道`Promise`的状态，这也说明`finally`内部的操作与状态无关。**`finally`本质上是`then`方法的特例**。下面是`finally`的实现：

```javascript
Promise.prototype.finally = function (callback) {
  let P = this.constructor;
  return this.then(
    (value) => P.resolve(callback()).then(() => value),
    (reason) =>
      P.resolve(callback()).then(() => {
        throw reason;
      })
  );
};
```

## Promise.all()

`Promise.all`可以将多个`Promise`实例包装成一个新的`Promise`实例。

```javascript
const p = Promise.all([p1, p2, p3]);
```

上面的代码`Promise.all()`方法接受一个数组作为参数，数组内的元素可以不是`Promise`实例，如果不是会先调用`Promise.resolve`方法进行转换。参数也可以不是数组，但必须具有`Iterator`接口，并且返回的每个成员都是`Promise`实例。
`p`的状态也是有两种状态：

- 只有 `p1、p2、p3` 的状态都变成 `fulfilled`，`p` 的状态才会变成 `fulfilled`，此时 `p1、p2、p3` 的返回值组成一个数组，传递给 `p` 的回调函数。
- 只要 `p1、p2、p3` 之中有一个被 `rejected`，`p` 的状态就变成 `rejected`，此时第一个被 `reject` 的实例的返回值，会传递给 `p` 的回调函数。
  注意：如果作为参数`Promise`实例自己定义了`catch`方法，那么他一旦被`rejected`,并不会触发`Promise.all()`的`catch`方法

```javascript
const p1 = new Promise((resolve, reject) => {
  resolve("hello");
})
  .then((result) => result)
  .catch((e) => e);

const p2 = new Promise((resolve, reject) => {
  throw new Error("报错了");
})
  .then((result) => result)
  .catch((e) => e);

Promise.all([p1, p2])
  .then((result) => console.log(result))
  .catch((e) => console.log(e));
// ["hello", Error: 报错了]
```

上面的代码`p1`会`resolved`,`p2`会先`rejected`,但是`p2`有自己的`catch`方法，改方法返回一个新的`Promise`实例，`p2`实际上指向的是这个新实例。该实例最后也是`resolved`状态，所以最后`Promise.all()`的两个参数都会`resolved`,最终执行`then`方法的回调。
如果`p2`没有自己的`catch`方法，就会调用`Promise.all()`的`catch`方法：

```javascript
const p1 = new Promise((resolve, reject) => {
  resolve("hello");
}).then((result) => result);

const p2 = new Promise((resolve, reject) => {
  throw new Error("报错了");
}).then((result) => result);

Promise.all([p1, p2])
  .then((result) => console.log(result))
  .catch((e) => console.log(e));
// Error: 报错了
```

## Promise.race()

`Promise.race`也是将多个`Promise`实例包装成一个新的`Promise`实例。

```javascript
const p = Promise.race([p1, p2, p3]);
```

上面的代码中，只要有一个实例改变状态，`p`的状态就随之改变，那个最先改变状态的实例的返回值就传递给`p`的回调函数。`Promise.race`也会将参数中非`Promise`实例进行转换。
下面的例子：如果在指定时间没有获得结果就将状态变为`rejected`

```javascript
const p = Promise.race([
  fetch("/resource-that-may-take-a-while"),
  new Promise(function (resolve, reject) {
    setTimeout(() => reject(new Error("request timeout")), 5000);
  }),
]);

p.then(console.log).catch(console.error);
```

## Promise.allSettled()

`Promise.allSettled()`方法接受一个数组作为参数，数组的每个成员都是一个 `Promise` 对象，并返回一个新的 `Promise` 对象。只有等到参数数组的所有 `Promise` 对象都发生状态变更（不管是 `fulfilled` 还是 `rejected`），返回的 `Promise` 对象才会发生状态变更。

```javascript
const resolved = Promise.resolve(42);
const rejected = Promise.reject(-1);

const allSettledPromise = Promise.allSettled([resolved, rejected]);

allSettledPromise.then(function (results) {
  console.log(results);
});
// [
//    { status: 'fulfilled', value: 42 },
//    { status: 'rejected', reason: -1 }
// ]
```

`Promise.allSettled()`返回的新的实例状态只会从`pending`变为`fulfilled`,不会变成`rejected`。状态改变后，它的回调函数会接收到一个数组，该数组的每个要素就是对应前面数组的每个`Promise`对象。数组成员格式如下：

```javascript
// 异步操作成功时
{status: 'fulfilled', value: value}

// 异步操作失败时
{status: 'rejected', reason: reason}
```

## Promise.any()

该方法接受一组`Promise`实例作为参数，包装成一个新的`Promise`实例返回。它只要有一个参数变成`fulfilled`状态，返回的实例就会变成`fulfilled`;如果所有参数实例变成`rejected`,包装的实例就会变成`rejected`。它与`race`方法的区别就是`any`需要等待所有的参数实例变成`rejected`才会结束。

```javascript
var resolved = Promise.resolve(42);
var rejected = Promise.reject(-1);
var alsoRejected = Promise.reject(Infinity);

Promise.any([resolved, rejected, alsoRejected]).then(function (result) {
  console.log(result); // 42
});

Promise.any([rejected, alsoRejected]).catch(function (results) {
  console.log(results.errors); // [-1, Infinity]
});
```

## Promise.resolve()

该方法可以将一个普通对象转换成`Promise`对象。

```javascript
Promise.resolve("foo");
// 等价于
new Promise((resolve) => resolve("foo"));
```

`Promise.resolve()`的参数有四种情况

- Promise 实例：不做任何修改，直接返回
- thenable 对象：`thenable`对象指具有`then`方法的对象：

```javascript
let thenable = {
  then: function (resolve, reject) {
    resolve(42);
  },
};
```

`Promise.resolve()`会将该对象转换成`Promise`对象，然后立即执行其`then`方法

```javascript
let thenable = {
  then: function (resolve, reject) {
    resolve(42);
  },
};

let p1 = Promise.resolve(thenable);
p1.then(function (value) {
  console.log(value); // 42
});
```

上面的代码`thenable`对象执行`then`方法后，`p1`状态变为`resolved`,从而立即执行最后的`then`方法的回调。

- 参数不是具有 `then()`方法的对象，或根本就不是对象：如果参数是一个原始值，或者是一个不具有 `then()`方法的对象，则 `Promise.resolve()`方法返回一个新的 `Promise` 对象，状态为 `resolved`。

```javascript
const p = Promise.resolve("Hello");

p.then(function (s) {
  console.log(s);
});
// Hello
```

上面代码生成一个新的 `Promise` 对象的实例 `p`。由于字符串 `Hello` 不属于异步操作（判断方法是字符串对象不具有 `then` 方法），返回 `Promise` 实例的状态从一生成就是 `resolved`，所以回调函数会立即执行。`Promise.resolve()`方法的参数，会同时传给回调函数。

- 不带有任何参数
  `Promise.resolve`方法调用时不带参数，会直接返回一个`resolved`状态的`Promise`对象。注意：**立即`resolve()`的 Promise 对象，是在本轮“事件循环”的结束时执行的，而不是在下一轮“事件循环”的开始时执行**

```javascript
setTimeout(function () {
  console.log("three");
}, 0);

Promise.resolve().then(function () {
  console.log("two");
});

console.log("one");

// one
// two
// three
```

## Promise.reject()

`Promise.reject()`方法会返回一个新的`Promise`实例，该实例的状态为`rejected`.`Promise.reject()`方法的参数，会原封不动地作为`reject`的理由，变成后续方法的参数.

```javascript
const p = Promise.reject("出错了");
// 等同于
const p = new Promise((resolve, reject) => reject("出错了"));

p.then(null, function (s) {
  console.log(s);
}); // 出错了
Promise.reject("出错了").catch((e) => {
  console.log(e === "出错了");
});
// true
```

## Promise.try()

在实际开发中，我们会遇到一种情况：对于函数`f`,无论它是同步函数还是异步函数，我们都想用`Promise`来处理它，因为这样我们可以用`then`来指定下一步流程，用`catch`处理抛出的错误。我们有下面几种处理方式：

1. `Promise.resolve().then(f)`:这种方法如果`f`是同步函数，他会在本轮**事件循环**的末尾执行，如下面的例子：

```javascript
const f = () => console.log("now");
Promise.resolve().then(f);
console.log("next");
// next
// now
```

同步代码`console.log("now")`变成了异步执行。

2. 用 async 函数：

```javascript
const f = () => console.log("now");
(async () => f())();
console.log("next");
// now
// next
```

上面代码中，第二行是一个立即执行的匿名函数，会立即执行里面的 `async` 函数，因此如果 `f` 是同步的，就会得到同步的结果；如果 `f` 是异步的，就可以用 `then` 指定下一步，就像下面的写法:

```javascript
(async () => f())()
.then(...)
.catch(...)
```

3. 使用 new Promise()

```javascript
const f = () => console.log("now");
(() => new Promise((resolve) => resolve(f())))();
console.log("next");
// now
// next
```

4. Promise.try

```javascript
Promise.try(() => database.users.get({id: userId}))
  .then(...)
  .catch(...)
```

上面的`database.users.get({id: userId})`返回一个`Promise`,它的可能会返回同步错误，如果不使用`Promise.try`我们就需要使用`try...catch`去处理。而事实上，Promise.try 就是模拟 try 代码块，就像 promise.catch 模拟的是 catch 代码块。

> [ECMAScript 6 入门 Promise 对象](https://es6.ruanyifeng.com/#docs/promise)
