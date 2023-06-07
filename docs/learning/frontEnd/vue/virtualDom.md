# vue 和 react 的核心--`Virtual Dom `

## 前言

无论大环境怎么变现在前端的最主流的框架还是`vue`和`react`这两大霸主。而这两个框架的核心都是`Virtual DOM`+`Diff`算法。

> Vue 和 React 通用流程：vue template/react jsx -> render 函数 -> 生成 VNode -> 当有变化时，新老 VNode diff -> diff 算法对比，并真正去更新真实 DOM。

本篇我们来主要探究核心之一--`Virtual Dom`。

## 真实 DOM 和其解析流程

在了解`vNode`之前我们先来认识真实`DOM`，直接上图
![virtualDom1](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/virtualDom1.png)
上面的图是`webkit`渲染引擎工作流程，所有的浏览器渲染引擎工作流程大致分为 5 步：创建`DOM`树-->创建`Style Rules`-->构建`Render`树-->布局`LayOut`-->绘制`Painting`。

1. 构建`DOM`树：用 HTML 分析器分析 `HTML` 元素，构建一颗 `DOM` 树；
2. 生成样式表：用 CSS 分析器，分析`CSS` 文件和元素上的 `inline` 样式，生成页面的样式表；
3. 构建`Render`树：将 DOM 树和样式表关联起来，构建一颗 `Render` 树（`Attachment`）。每个 `DOM` 节点都有 `attach` 方法，接受样式信息，返回一个 `render` 对象（也叫 `renderer`）,这些 `render` 对象最终会被构建成一棵 `Render` 树；
4. 确定节点坐标：根据 `Render` 树结构，为每个 `Render` 树上的节点确定一个在显示屏上出现的精确位置；
5. 绘制页面：根据 `Render` 树和节点吓死你坐标，然后调用每个节点的 `paint` 方法，将它们绘制出来。

   tips:

- 构建 DOM 树是一个渐进过程，为了达到更好的用户体验，渲染引擎会尽快将内容显示在屏幕上，他不必等到整个`HTML`文档解析完成之后才开始构建`render`树和布局
- `Render` 树是 `DOM` 树和 `CSS` 样式表这三个过程在实际进行的时候并不是完全独立的，而是会有交叉一边加载一边解析并且同时渲染
- css 的解析是从右往左逆向解析，嵌套标签越多，解析越慢
- 用我们传统的开发模式，原生操作 `DOM` 时，浏览器会从构建 `DOM` 树开始从头到尾执行一遍流程。而 `DOM` 是十分庞大的，里面包含很多的属性，所以说操作 `DOM` 的代价是十分昂贵的。举一个网上常见的例子：当你一次操作要更新 10 个 `DOM` 节点，浏览器在收到第一个更新请求后就会马上执行操作，最终就执行 10 次操作。而通过 `VNode`,同样更新 10 个节点，`VNode` 不会立即操作 `DOM`,而是将 10 次更新的内容记录保存在本地的一个 `js` 对象上，最后这个 `js` 对象一次性 `attach` 到 `DOM` 树上。

## 什么是`Virtual Dom `

`Virtual DOM`本质上及时一个 `Javascript` 对象，是对真是 `DOM` 的描述。`VNode` 可以看作是一个模拟 `DOM` 树的 `js` 树，其主要是通过 `VNode` 实现一个无状态组件，当组件状态发生更新时，触发 `Virtual DOM` 数据的变化，然后通过 `Virtual DOM` 和真是 `DOM` 的比对去更新真是 `DOM`。所以我们可以认为 `Virtual DOM` 是真实 `DOM` 的缓存。

### 优点

- **跨平台与分层设计**： `Virtual Dom ` 本质上是 `JavaScript  `对象，而真实 DOM 与平台强相关，这样`Virtual Dom ` 就具有了分层设计、跨平台以及 `SSR ` 等特性。`Virtual Dom ` 抽象了原来的渲染过程，不再局限于浏览器的 `DOM ` 还可以是安卓和 `iOS ` 的原生组件、小程序甚至是各种 `GUI `。
- **以最小的代价更新变化的视图**：`Virtual Dom ` 能通过 patch 准确地转换成真实 `DOM `，这样我们就能通过 `diff ` 算法更方便的进行 `DOM ` 更新
- **保证性能下限**：`Virtual Dom ` 需要视频任何上层 `API ` 可能产生的操作，所以它必须是普适的，这也决定它的性能并不是最优的；但是比起直接无脑操作 `DOM ` 的性能要很多，它至少可以保证你不用手动优化 `DOM ` 操作的情况下依然拥有不错的性能体验
- **无需手动操作 DOM**:有了`Virtual Dom `，我们可以不再手动操作 `DOM `,只要写好 `view-model ` 的代码逻辑，框架会根据`Virtual Dom ` 和数据双向绑定帮我们去更新视图
- **组件高度抽象化**： `Vue2.x ` 通过`Virtual Dom ` 把渲染过程抽象化，不再依赖 `HTML ` 解析器进行模版解析，可以进行更多的 `AOT `(预编译)提高效率，运行体积压缩，运行效率提高（这些都是 `AOT ` 带来的优势）。 `Virtual Dom ` 的优势不在于单次的操作（首次渲染大量 `DOM ` 时，由于多了一层`Virtual Dom ` 的计算，会比 `innerHTML ` 插入慢），而是在大量、频繁的数据更新下能对视图进行合理、高效的更新，当然这需要一套高效的 `diff ` 算法来配合。

### 缺点

- 无法进行极致优化：因为其普适性，在一些性能要求 极高的应用中还是难以胜任
- 虽然可以保证触发更新组件的最小化，但在单个组件内部还是要遍历该组件整个 `Virtual DOM` 树
- 在只有少量动态节点变化的情况下，这些遍历是性能的浪费
- 传统 `Virtual DOM` 的性能跟模版大小正相关，跟动态节点的数量无关

### 虚拟 DOM 的基本原理

- 用 `Javascript` 对象模拟真实 `DOM`；
- 用 `diff` 算法比对新旧虚拟 `DOM` 树的差异
- `patch` 算法将新旧虚拟 `DOM` 树应用到真实 `DOM` 上

## Vue 源码 Virtual-DOM 简析

上面我们说到虚拟 `DOM` 就是用 `js` 描述一个真实的 `DOM` 节点。我们通过 `Vue` 中的`VNode` 的定义、`diff`、`patch` 等过程来进行学习

### VNode 类

`VNode` 类定义在`src/core/vdom/vnode.js`,`Vue` 中的 `Virtual DOM` 借鉴了一个开源库`snabbdom`的实现，然后加入了一些 `Vue` 自身的特性。

```javascript
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  functionalContext: Component | void; // only for functional component root nodes
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?

  constructor(
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions
  ) {
    /*当前节点的标签名*/
    this.tag = tag;
    /*当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息*/
    this.data = data;
    /*当前节点的子节点，是一个数组*/
    this.children = children;
    /*当前节点的文本*/
    this.text = text;
    /*当前虚拟节点对应的真实dom节点*/
    this.elm = elm;
    /*当前节点的名字空间*/
    this.ns = undefined;
    /*编译作用域*/
    this.context = context;
    /*函数化组件作用域*/
    this.functionalContext = undefined;
    /*节点的key属性，被当作节点的标志，用以优化*/
    this.key = data && data.key;
    /*组件的option选项*/
    this.componentOptions = componentOptions;
    /*当前节点对应的组件的实例*/
    this.componentInstance = undefined;
    /*当前节点的父节点*/
    this.parent = undefined;
    /*简而言之就是是否为原生HTML或只是普通文本，innerHTML的时候为true，textContent的时候为false*/
    this.raw = false;
    /*静态节点标志*/
    this.isStatic = false;
    /*是否作为跟节点插入*/
    this.isRootInsert = true;
    /*是否为注释节点*/
    this.isComment = false;
    /*是否为克隆节点*/
    this.isCloned = false;
    /*是否有v-once指令*/
    this.isOnce = false;
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next https://github.com/answershuto/learnVue*/
  get child(): Component | void {
    return this.componentInstance;
  }
}
```

虽然 `VNode` 的属性很多，但是我们需要了解的只有一下几个核心属性：

- `tag`:代表这个 `VNode` 的标签属性
- `data`:属性包含了最后渲染成真是 `DOM` 节点后节点上的 `class`、`attribute`、`style` 以及绑定的事件
- `context`:指向 `Vue` 实例
- `children`:`VNode` 的子节点
- `text`:文本属性
- `ele`:这个 `VNode` 对应的真实节点
- `key`:`VNode` 的标记在 `diff` 过程中可以提高对比效率

  `VNode` 的类型有以下几种：

- 注释节点
- 文本节点
- 元素节点
- 组件节点
- 函数式组件节点
- 克隆节点
  在我们的视图渲染之前 ，框架会把 `template` 模版先编译成 `VNode` 保存下来，等到数据发生变化需要重新渲染的时候，框架会把新生成的 `VNode` 对象与之前缓存的旧的 `VNode` 对象进行对比找出差异，然后有差异的 `VNode` 对应的真实 `DOM` 节点就是需要重新渲染的节点，然后框架对应创建出真实的 `DOM` 节点插入到视图中完成更新。

## `_render`

Vue 的`_render` 方法是实例的一个私有方法，它用来吧实例渲染成一个虚拟 node

```javascript
 Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options
    let vnode
    try {
      // 省略一系列代码
      currentRenderingInstance = vm
      // 调用 createElement 方法来返回 vnode
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      handleError(e, vm, `render`){}
    }
    // set parent
    vnode.parent = _parentVnode
    console.log("vnode...:",vnode);
    return vnode
  }
```

## createElement

`Vue` 是通过 `createElement` 方法来生成 `VNode`

```javascript
export function createElement(
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children;
    children = data;
    data = undefined;
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE;
  }
  return _createElement(context, tag, data, children, normalizationType);
}
```

阅读上面的源码我们可以看到 `createElement` 方法其实是对`_createElement` 方法的封装，对参数进行了类型判断

```typescript
export function _createElement(
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== "production" &&
      warn(
        `Avoid using observed data object as vnode data: ${JSON.stringify(
          data
        )}\n` + "Always create fresh vnode data objects in each render!",
        context
      );
    return createEmptyVNode();
  }
  // object syntax in v-bind
    if (isDef(data) && isDef(data.is)) {
        tag = data.is
    }
    if (!tag) {
        // in case of component :is set to falsy value
        return createEmptyVNode()
    }
    ...
    // support single function children as default scoped slot
    if (Array.isArray(children) &&
        typeof children[0] === 'function'
    ) {
        data = data || {}
        data.scopedSlots = { default: children[0] }
        children.length = 0
    }
    if (normalizationType === ALWAYS_NORMALIZE) {
        children = normalizeChildren(children)
    } else if ( === SIMPLE_NORMALIZE) {
        children = simpleNormalizeChildren(children)
    }
 // 创建VNode
    ...
}
```

- `context`:表示 `VNode` 的上下文环境,是 `Component` 类型
- `tag`:表示标签，可以是字符串或者是 `Component` 类型
- `data`:表示 `VNode 的数据，它是一个 `VNodeData` 类型
- `children`:表示当前 `VNode` 的子节点，它是任意类型
- `normalizationType`：表示子节点规范的类型，类型不同规范的范围也就不同，主要是参考 `render` 函数是编译生成还是用户手写的
  根据 `normalizationType` 的类型，`children` 有不同的定义

```typescript
if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
} else if ( === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
}
```

`simpleNormalizeChildren` 是在 `render` 是编译生成的时候调用的
`normalizeChildren` 的调用场景有以下两种：

1. `render` 是用户手写的
2. 编译 `slot`,`v-for` 的时候产生嵌套数组
   它们都是对 `children` 的规范化（使 `children` 变成一个类型为 `VNode` 的数组,因为 `children` 的类型是任意类型）
   规范化后框架就会去创建 `VNode`

```typescript
let vnode, ns;
// 对tag进行判断
if (typeof tag === "string") {
  let Ctor;
  ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag);
  if (config.isReservedTag(tag)) {
    // 如果是内置的节点，则直接创建一个普通VNode
    vnode = new VNode(
      config.parsePlatformTagName(tag),
      data,
      children,
      undefined,
      undefined,
      context
    );
  } else if (
    isDef((Ctor = resolveAsset(context.$options, "components", tag)))
  ) {
    // component
    // 如果是component类型，则会通过createComponent创建VNode节点
    vnode = createComponent(Ctor, data, context, children, tag);
  } else {
    vnode = new VNode(tag, data, children, undefined, undefined, context);
  }
} else {
  // direct component options / constructor
  vnode = createComponent(tag, data, context, children);
}
```

`createComponent` 也是用来创建 `VNode` 的

```typescript
export function createComponent(
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return;
  }
  // 构建子类构造函数
  const baseCtor = context.$options._base;

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor);
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== "function") {
    if (process.env.NODE_ENV !== "production") {
      warn(`Invalid Component definition: ${String(Ctor)}`, context);
    }
    return;
  }

  // async component
  let asyncFactory;
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor;
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context);
    if (Ctor === undefined) {
      return createAsyncPlaceholder(asyncFactory, data, context, children, tag);
    }
  }

  data = data || {};

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor);

  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data);
  }

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag);

  // functional component
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children);
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  const listeners = data.on;
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn;

  if (isTrue(Ctor.options.abstract)) {
    const slot = data.slot;
    data = {};
    if (slot) {
      data.slot = slot;
    }
  }

  // 安装组件钩子函数，把钩子函数合并到data.hook中
  installComponentHooks(data);

  //实例化一个VNode返回。组件的VNode是没有children的
  const name = Ctor.options.name || tag;
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ""}`,
    data,
    undefined,
    undefined,
    undefined,
    context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  );
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode);
  }

  return vnode;
}
```

它主要有三个步骤：

1. 构造子类构造函数 `Ctor`
2. `installComponentHooks` 安装组件钩子函数
3. 实例化 `VNode`

## diff 和 patch

`Vue` 源码实例化了一个 `watcher`,它被添加到了在模版当中所绑定变量的依赖中，一旦 `model` 中的响应式的数据发生变化，这些响应式的数据所维护的 `dep` 数组便会调用 `dep.notify()`方法完成所有依赖遍历执行的工作，其中包括视图的更新，即 `updateComponent` 方法的调用。

```javascript
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el;
  // 省略一系列其它代码
  let updateComponent;
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== "production" && config.performance && mark) {
    updateComponent = () => {
      // 生成虚拟 vnode
      const vnode = vm._render();
      // 更新 DOM
      vm._update(vnode, hydrating);
    };
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating);
    };
  }

  // 实例化一个渲染Watcher，在它的回调函数中会调用 updateComponent 方法
  new Watcher(
    vm,
    updateComponent,
    noop,
    {
      before() {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, "beforeUpdate");
        }
      },
    },
    true /* isRenderWatcher */
  );
  hydrating = false;

  return vm;
}
```

完成视图的更新工作事实上就是调用 `vm._update` 方法，这个方法接收的第一个参数就是刚生成的 `VNode`

```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this;
  const prevEl = vm.$el;
  const prevVnode = vm._vnode;
  const restoreActiveInstance = setActiveInstance(vm);
  vm._vnode = vnode;
  if (!prevVnode) {
    // 第一个参数为真实的node节点，则为初始化
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
  } else {
    // 如果需要diff的prevVnode存在，那么对prevVnode和vnode进行diff
    vm.$el = vm.__patch__(prevVnode, vnode);
  }
  restoreActiveInstance();
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null;
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm;
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el;
  }
};
```

在这个方法当中最为关键的就是`vm._patch_`方法,主要完成了 `prevVnode` 和 `vnode` 的 `diff` 过程并根据需要操作的 `vdom` 节点打 patch,最后生成新的真实 `dom` 节点并完成视图的更新工作

```javascript
function patch (oldVnode, vnode, hydrating, removeOnly) {
    ......
    if (isUndef(oldVnode)) {
      // 当oldVnode不存在时，创建新的节点
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      // 对oldVnode和vnode进行diff，并对oldVnode打patch
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      }
	......
  }
}
```

在 `patch` 方法中，当 `oldVnode` 不存在时会创建新的节点；如果已经存在 `oldVNode`，就会对 `oldVNode` 和 `vnode` 进行 `diff` 和 `patch`。其中 `patch` 过程会调用 `sameVnode` 方法对传入的两个 `vnode` 进行基本属性比较，如果基本属性相同会认为两个 `vnode` 只是局部发生更新，然后才会对这两个 `vnode` 进行 `diff` 比较；这两个 `vnode` 的基本属性不同，就跳过 `diff` 直接创建一个新的真实 `DOM`,删除旧的 `DOM` 节点。

```javascript
function sameVnode(a, b) {
  return (
    a.key === b.key &&
    a.tag === b.tag &&
    a.isComment === b.isComment &&
    isDef(a.data) === isDef(b.data) &&
    sameInputType(a, b)
  );
}
```

`diff` 过程中主要是通过调用 `patchVnode` 方法进行的:

```javascript
  function patchVnode (oldVnode, vnode, insertedVnodeQueue, ownerArray, index, removeOnly) {
    ......
    const elm = vnode.elm = oldVnode.elm
    const oldCh = oldVnode.children
    const ch = vnode.children
    // 如果vnode没有文本节点
    if (isUndef(vnode.text)) {
      // 如果oldVnode的children属性存在且vnode的children属性也存在
      if (isDef(oldCh) && isDef(ch)) {
        // updateChildren，对子节点进行diff
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        // 如果oldVnode的text存在，那么首先清空text的内容,然后将vnode的children添加进去
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        // 删除elm下的oldchildren
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        // oldVnode有子节点，而vnode没有，那么就清空这个节点
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      // 如果oldVnode和vnode文本属性不同，那么直接更新真是dom节点的文本元素
      nodeOps.setTextContent(elm, vnode.text)
    }
    ......
  }
```

上面的代码流程如下：

- 首先进行文本节点的判断，若 `oldVnode.text!==vnode.text`,就直接进行文本节点的替换；
- 在 `vnode` 没有文本节点的情况下，进入子节点的 `diff`;
- 当 `oldCH` 和 `ch` 都存在且不相同的情况下，调用 `updateChildren` 对子节点进行 `diff`
- 如果 `oldCh 不存在，`ch`存在，首先情况`oldVnode`的文本节点，同时调用`addVnodes`方法将`ch`添加到`elm`真实的`dom` 节点中
- 若 `oldCh` 存在，`ch` 不存在就直接删除 `elm` 下的 `oldCh` 节点
- 若 `oldVnode` 有文本节点，而 `vnode` 没有，就清空对应的文本节点
  子节点的 `diff` 主要是通过 `updateChildren` 方法，下面是它的源码

```javascript
function updateChildren(
  parentElm,
  oldCh,
  newCh,
  insertedVnodeQueue,
  removeOnly
) {
  // 为oldCh和newCh分别建立索引，为之后遍历的依据
  let oldStartIdx = 0;
  let newStartIdx = 0;
  let oldEndIdx = oldCh.length - 1;
  let oldStartVnode = oldCh[0];
  let oldEndVnode = oldCh[oldEndIdx];
  let newEndIdx = newCh.length - 1;
  let newStartVnode = newCh[0];
  let newEndVnode = newCh[newEndIdx];
  let oldKeyToIdx, idxInOld, vnodeToMove, refElm;

  // 直到oldCh或者newCh被遍历完后跳出循环
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (isUndef(oldStartVnode)) {
      oldStartVnode = oldCh[++oldStartIdx]; // Vnode has been moved left
    } else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx];
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      patchVnode(
        oldStartVnode,
        newStartVnode,
        insertedVnodeQueue,
        newCh,
        newStartIdx
      );
      oldStartVnode = oldCh[++oldStartIdx];
      newStartVnode = newCh[++newStartIdx];
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      patchVnode(
        oldEndVnode,
        newEndVnode,
        insertedVnodeQueue,
        newCh,
        newEndIdx
      );
      oldEndVnode = oldCh[--oldEndIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldStartVnode, newEndVnode)) {
      // Vnode moved right
      patchVnode(
        oldStartVnode,
        newEndVnode,
        insertedVnodeQueue,
        newCh,
        newEndIdx
      );
      canMove &&
        nodeOps.insertBefore(
          parentElm,
          oldStartVnode.elm,
          nodeOps.nextSibling(oldEndVnode.elm)
        );
      oldStartVnode = oldCh[++oldStartIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldEndVnode, newStartVnode)) {
      // Vnode moved left
      patchVnode(
        oldEndVnode,
        newStartVnode,
        insertedVnodeQueue,
        newCh,
        newStartIdx
      );
      canMove &&
        nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
      oldEndVnode = oldCh[--oldEndIdx];
      newStartVnode = newCh[++newStartIdx];
    } else {
      if (isUndef(oldKeyToIdx))
        oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key]
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);
      if (isUndef(idxInOld)) {
        // New element
        createElm(
          newStartVnode,
          insertedVnodeQueue,
          parentElm,
          oldStartVnode.elm,
          false,
          newCh,
          newStartIdx
        );
      } else {
        vnodeToMove = oldCh[idxInOld];
        if (sameVnode(vnodeToMove, newStartVnode)) {
          patchVnode(
            vnodeToMove,
            newStartVnode,
            insertedVnodeQueue,
            newCh,
            newStartIdx
          );
          oldCh[idxInOld] = undefined;
          canMove &&
            nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm);
        } else {
          // same key but different element. treat as new element
          createElm(
            newStartVnode,
            insertedVnodeQueue,
            parentElm,
            oldStartVnode.elm,
            false,
            newCh,
            newStartIdx
          );
        }
      }
      newStartVnode = newCh[++newStartIdx];
    }
  }
  if (oldStartIdx > oldEndIdx) {
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm;
    addVnodes(
      parentElm,
      refElm,
      newCh,
      newStartIdx,
      newEndIdx,
      insertedVnodeQueue
    );
  } else if (newStartIdx > newEndIdx) {
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
  }
}
```

在开始遍历 `diff` 之前，首先给 `oldCh` 和 `newCh` 分别分配一个 `startIndex` 和 `endIndex` 来作为遍历的索引，当 `oldCh` 或者 `newCh` 遍历完后（就是 `oldCh` 或者 `newCh` 的 `startIndex>=endIndex`）,就停止 `oldCh` 和 `newOld` 的 `diff` 过程。（这个过程比较复杂也比较重要，我会单开一篇来单独讲它）

在上面的源码中我们可以看到 `nodeOps` 相关的方法对真实 `DOM` 进行操作
`_update` 会将新旧两个 `Vnode 进行 patch,得出两个 VNode 最小差异，然后将这些差异渲染到视图上。整个 `patch 就是干三件事：

- 创建节点：新的 `VNode` 有而旧的没有，就在旧的 `Vnode` 中创建
- 删除节点：新的没有而旧的有，就从旧的中删除
- 更新节点：新的 `VNode` 和旧的 `VNode` 都有，就以新的为准进行更新

### createElm

`VNode` 类描述了六种类型的节点，但是只有 3 种类型的节点能够被创建并插入到 `DOM` 中，分别是：元素节点、文本节点、注释节点

```javascript
// 源码位置: /src/core/vdom/patch.js
function createElm(vnode, parentElm, refElm) {
  const data = vnode.data;
  const children = vnode.children;
  const tag = vnode.tag;
  if (isDef(tag)) {
    vnode.elm = nodeOps.createElement(tag, vnode); // 创建元素节点
    createChildren(vnode, children, insertedVnodeQueue); // 创建元素节点的子节点
    insert(parentElm, vnode.elm, refElm); // 插入到DOM中
  } else if (isTrue(vnode.isComment)) {
    vnode.elm = nodeOps.createComment(vnode.text); // 创建注释节点
    insert(parentElm, vnode.elm, refElm); // 插入到DOM中
  } else {
    vnode.elm = nodeOps.createTextNode(vnode.text); // 创建文本节点
    insert(parentElm, vnode.elm, refElm); // 插入到DOM中
  }
}
```

```javascript
export function createElementNS(namespace: string, tagName: string): Element {
  return document.createElementNS(namespaceMap[namespace], tagName);
}

export function createTextNode(text: string): Text {
  return document.createTextNode(text);
}

export function createComment(text: string): Comment {
  return document.createComment(text);
}

export function insertBefore(
  parentNode: Node,
  newNode: Node,
  referenceNode: Node
) {
  parentNode.insertBefore(newNode, referenceNode);
}

export function removeChild(node: Node, child: Node) {
  node.removeChild(child);
}
```

```javascript
function removeNode(el) {
  const parent = nodeOps.parentNode(el); // 获取父节点
  if (isDef(parent)) {
    nodeOps.removeChild(parent, el); // 调用父节点的removeChild方法
  }
}
```

## 总结

![virtualDom2](https://raw.githubusercontent.com/justingcode/my-diary/main/docs/media/img/virtualDom2.png)

- 当数据发生变化，订阅者 `watcher` 会调用 `patch` 给真实 `DOM` 打补丁
- 通过 `isSameVnode` 进行判断，相同就调用 `patchVnode`
- `patchVnode` 进行下面的操作：
  - 找到对应真实的 `DOM`,成为 `el`
  - 如果都有文本节点切不相同，将 `el` 文本节点设置为 `Vnode` 的文本节点
  - 如果 `oldVnode` 有子节点而 `VNode` 没有，就删除 `el` 子节点
  - 如果 `oldVnode` 没有子节点儿 `VNode` 有，就将 `VNode` 子节点真实化添加到 `el`
  - 如果都有子节点就执行 `updateChildren 比较子节点
- `updateChildren` 的操作
  - 设置新旧 `VNode` 的头尾指针
  - 新旧的头尾指针循环向中间靠拢比较，调用 `patchVnode` 进行 `patch`、调用 `createElm` 创建新节点，从哈希表寻找 `key` 一致的 `VNode` 节点再分情况操作

> [个人理解 Vue 和 React 区别](https://lq782655835.github.io/blogs/vue/diff-vue-vs-react.html)

> [深入剖析：Vue 核心之`Virtual Dom `](https://juejin.cn/post/6844903895467032589)

> [虚拟 Dom ](https://jonny-wei.github.io/blog/vue/vue/vue-diff.html#%E8%99%9A%E6%8B%9F-dom)

> [虚拟 DOM 的理解与总结 ](https://www.cnblogs.com/smileZAZ/p/16478386.html)

> [面试官：什么是虚拟 DOM？如何实现一个虚拟 DOM？](https://cloud.tencent.com/developer/article/old/1794275)
