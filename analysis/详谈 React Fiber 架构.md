## 前言

2016 年都已经透露出来的概念，这都 9102 年了，我才开始写 Fiber 的文章，表示惭愧呀。不过现在好的是关于 Fiber 的资料已经很丰富了，在写文章的时候参考资料比较多，比较容易深刻的理解。

React 作为我最喜欢的框架，没有之一，我愿意花很多时间来好好的学习他，我发现对于学习一门框架会有四种感受，刚开始没使用过，可能有一种很神奇的感觉；然后接触了，遇到了不熟悉的语法，感觉这是什么垃圾东西，这不是反人类么；然后当你熟悉了之后，真香，设计得挺好的，这个时候它已经改变了你编程的思维方式了；再到后来，看过他的源码，理解他的设计之后，设计得确实好，感觉自己也能写一个的样子。

所以我**今年**(对，没错，就是一年)就是想完全的学透 React，所以开了一个 **Deep In React** 的系列，把一些新手在使用 API 的时候不知道为什么的点，以及一些为什么有些东西要这么设计写出来，与大家共同探讨 React 的奥秘。

我的思路是自上而下的介绍，先理解整体的 Fiber 架构，然后再细挖每一个点，所以这篇文章主要是谈 Fiber 架构的。

## 介绍

在详细介绍 Fiber 之前，先了解一下 Fiber 是什么，以及为什么 React 团队要话两年时间重构协调算法。

在讨论为什么要用 Fiber 架构的话，得先了解之前的协调算法有什么问题。

### React 的核心思想

**内存中维护一颗虚拟DOM树，数据变化时（setState），自动更新虚拟 DOM，得到一颗新树，然后 Diff 新老虚拟 DOM 树，找到有变化的部分，得到一个 Change(Patch)，将这个 Patch 加入队列，最终批量更新这些 Patch 到 DOM 中**。

### React 16 之前的不足

首先我们了解一下 React 的工作过程，当我们通过`render() `和 `setState()` 进行组件渲染和更新的时候，React 主要有两个阶段：

**调和阶段(Reconciler)：**官方解释[点我查看](https://zh-hans.reactjs.org/docs/reconciliation.html)。React 会自顶向下通过递归，遍历新数据生成新的 Virtual DOM，然后通过 Diff 算法，找到需要变更的元素(Patch)，放到更新队列里面去。

**渲染阶段(Renderer)**：遍历更新队列，通过调用宿主环境的API，实际更新渲染对应元素。宿主环境，比如 DOM、Native、WebGL 等。

在协调阶段阶段，由于是采用的递归的遍历方式，这种也被成为 Stack Reconciler，主要是为了区别 Fiber Reconciler 取的一个名字。这种方式有一个特点：一旦任务开始进行，就无法中断，那么 js 将一直占用主线程， 一直要等到整棵 Virtual DOM 树计算完成之后，才能把执行权交给渲染引擎，那么这就会导致一些用户交互、动画等任务无法立即得到处理，就会有卡顿，非常的影响用户体验。

### 如何解决之前的不足

> 之前的问题主要的问题是任务一旦执行，就无法中断，js 线程一直占用主线程，导致卡顿。

可能有些接触前端不久的不是特别理解上面为什么 js 一直占用主线程就会卡顿，我这里还是简单的普及一下。

#### 浏览器每一帧都需要完成哪些工作？

页面是一帧一帧绘制出来的，当每秒绘制的帧数（FPS）达到 60 时，页面是流畅的，小于这个值时，用户会感觉到卡顿。

1s 60 帧，所以每一帧分到的时间是 1000/60 ≈ 16 ms。所以我们书写代码时力求不让一帧的工作量超过 16ms。

![image-20190603163205451](http://imgs.taoweng.site/2019-06-03-083206.png)

*浏览器一帧内的工作*



通过上图可看到，一帧内需要完成如下六个步骤的任务：

- 处理用户的交互
- JS 解析执行
- 帧开始。窗口尺寸变更，页面滚去等的处理
- rAF(requestAnimationFrame)
- 布局
- 绘制

如果这六个步骤中，任意一个步骤所占用的时间过长，总时间超过 16ms 了之后，用户也许就能看到卡顿。

而在上一小节提到的**调和阶段**花的时间过长，也就是 js 执行的时间过长，那么就有可能在用户有交互的时候，本来应该是渲染下一帧了，但是在当前一帧里还在执行 JS，就导致用户交互不能麻烦得到反馈，从而产生卡顿感。

#### 解决方案

**把渲染更新过程拆分成多个子任务，每次只做一小部分，做完看是否还有剩余时间，如果有继续下一个任务；如果没有，挂起当前任务，将时间控制权交给主线程，等主线程不忙的时候在继续执行。**这种策略叫做 Cooperative Scheduling（合作式调度），操作系统常用任务调度策略之一。

> **补充知识**，操作系统常用任务调度策略：先来先服务（FCFS）调度算法、短作业（进程）优先调度算法（SJ/PF）、最高优先权优先调度算法（FPF）、高响应比优先调度算法（HRN）、时间片轮转法（RR）、多级队列反馈法。

这个方案看似确实不错，但是怎么实现可能会遇到几个问题：

- 如何拆分成子任务？
- 一个子任务多大合适？
- 怎么判断是否还有剩余时间？
- 有剩余时间怎么去调度应该执行哪一个任务？
- 没有剩余时间之前的任务怎么办？

接下里整个 Fiber 架构就是来解决这些问题的。

## 什么是 Fiber

为了解决之前提到解决方案遇到的问题，提出了以下几个目标：

- 暂停工作，稍后再回来。
- 为不同类型的工作分配优先权。
- 重用以前完成的工作。
- 如果不再需要，则中止工作。

为了做到这些，我们首先需要一种方法将任务分解为单元。从某种意义上说，这就是 Fiber，Fiber 代表一种**工作单元**。

但是仅仅是分解为单元也无法做到中断任务，因为函数调用栈就是这样，每个函数为一个工作，每个工作被称为**堆栈帧**，它会一直工作，直到堆栈为空，无法中断。

所以我们需要一种增量渲染的调度，那么就需要重新实现一个堆栈帧的调度，这个堆栈帧可以按照自己的调度算法执行他们。另外由于这些堆栈是可以自己控制的，所以可以加入并发或者错误边界等功能。

因此 Fiber 就是重新实现的堆栈帧，本质上 Fiber 也可以理解为是一个**虚拟的堆栈帧**，将可中断的任务拆分成多个子任务，通过按照优先级来自由调度子任务，分段更新，从而将之前的同步渲染改为异步渲染。

所以我们可以说 Fiber 是一种数据结构(堆栈帧)，也可以说是一种解决可中断的调用任务的一种解决方案，它的特性就是**时间分片(time slicing)**和**暂停(supense)**。

> 如果了解**协程**的可能会觉得 Fiber 的这种解决方案，跟协程有点像(区别还是很大的)，是可以中断的，可以控制执行顺序。在 JS 里的 generator 其实就是一种协程的使用方式，不过颗粒度更小，可以控制函数里面的代码调用的顺序，也可以中断。

## Fiber 是如何工作的

1. `ReactDOM.render()` 和 `setState` 的时候开始创建更新。
2. 将创建的更新加入任务队列，等待调度。
3. 在 requestIdleCallback 空闲时执行任务。
4. 从根节点开始遍历 Fiber Node，并且构建 WokeInProgress Tree。
5. 生成 effectList。
6. 根据 EffectList 更新 DOM。

下面是一个简化的流程图：

![image-20190607172525729](http://ww3.sinaimg.cn/large/006tNc79ly1g3spwcg5jnj312c0lk151.jpg)

### Fiber Node

首先要谈到的就是基础的一个数据结构：FIber Node，承载了非常关键的上下文信息，可以说是贯彻整个创建和更新的流程，接下来就介绍一下这个 Fiber Node 所承载的信息。

```javascript
{
  // 组件相关
  tag: WorkTag, // 标记不同的组件类型
  key: null | string, // ReactElement里面的key
  elementType: any,// ReactElement.type，也就是我们调用`createElement`的第一个参数
  // The resolved function/class/ associated with this fiber.
  // 异步组件resolved之后返回的内容，一般是`function`或者`class`
  type: any,
    
  // The local state associated with this fiber.
  // 跟当前Fiber相关本地状态（比如浏览器环境就是DOM节点）
  stateNode: any,
  // 指向他在Fiber节点树中的`parent`，用来在处理完这个节点之后向上返回
  return: Fiber | null,
  // 单链表树结构
  // 指向自己的第一个子节点
  child: Fiber | null,
  // 指向自己的兄弟结构
  // 兄弟节点的return指向同一个父节点
  sibling: Fiber | null,
  index: number,

  // ref属性
  ref: null | (((handle: mixed) => void) & {_stringRef: ?string}) | RefObject,

  // 更新相关
  pendingProps: any,  // 新的变动带来的新的props
  memoizedProps: any,  // 上一次渲染完成之后的props
  updateQueue: UpdateQueue<any> | null,  // 该Fiber对应的组件产生的Update会存放在这个队列里面
  memoizedState: any, // 上一次渲染的时候的state
  firstContextDependency: ContextDependency<mixed> | null,// 一个列表，存放这个Fiber依赖的context
    
  // Scheduler 相关
  expirationTime: ExpirationTime,  // 代表任务在未来的哪个时间点应该被完成，不包括他的子树产生的任务
  // 快速确定子树中是否有不在等待的变化
  childExpirationTime: ExpirationTime,

  // 用来描述当前Fiber和他子树的`Bitfield`
  // 共存的模式表示这个子树是否默认是异步渲染的
  // Fiber被创建的时候他会继承父Fiber
  // 其他的标识也可以在创建的时候被设置
  // 但是在创建之后不应该再被修改，特别是他的子Fiber创建之前
  mode: TypeOfMode,
    
 // 在Fiber树更新的过程中，每个Fiber都会有一个跟其对应的Fiber
  // 我们称他为`current <==> workInProgress`
  // 在渲染完成之后他们会交换位置
  alternate: Fiber | null,

  // Effect 相关的
  effectTag: SideEffectTag, // 用来记录Side Effect
  nextEffect: Fiber | null, // 单链表用来快速查找下一个side effect
  firstEffect: Fiber | null,  // 子树中第一个side effect
  lastEffect: Fiber | null, // 子树中最后一个side effect
};
```

### Fiber Reconciler

Fiber Reconciler 是 React 里的调和器，这也是任务调度完成之后，如何去执行每个任务，如何去更新每一个节点的过程。

> 如果任务被打断了怎么办？

reconcile过程分为2个阶段（phase）：

1. （可中断）render/reconciliation 通过构造 WorkInProgress Tree 得出 Change
2. （不可中断）commit 应用这些DOM change

render 阶段可以理解为就是 Diff 的过程，得出 Change(Effect List)，会执行声明如下的声明周期方法：

- [UNSAFE_]componentWillMount（弃用）

- [UNSAFE_]componentWillReceiveProps（弃用）

- getDerivedStateFromProps

- shouldComponentUpdate

- [UNSAFE_]componentWillUpdate（弃用）

- render

而 commit 阶段可以理解为就是将 Diff 的结果反映到真实 DOM 的过程，会执行如下的声明周期方法：

- getSnapshotBeforeUpdate
- componentDidMount
- componentDidUpdate
- componentWillUnmount

由于 render 阶段是可中断的，一旦中断之后恢复的时候又会重新执行，所以很可能 render 阶段的生命周期方法会被多次调用。

> P.S：注意区别 reconciler、reconcile 和 reconciliation，reconciler 是调和器，是一个名词；reconcile 是调和器调和的动作，是一个动词；而 reconciliation 只是 reconcile 过程的第一个阶段。

### Fiber Tree 和 WorkInProgress Tree

> 在第一次渲染之后，React 最终得到一个 Fiber 树，它反映了用于渲染 UI 的应用程序的状态。这棵树通常被称为 **current 树（当前树）**。当 React 开始处理更新时，它会构建一个所谓的 **workInProgress 树（工作过程树）**，它反映了要刷新到屏幕的未来状态。

React 在 render 第一次渲染时，会通过 React.createElement 创建一颗 Element 树，可以称之为 Virtual DOM Tree，由于要记录上下文信息，加入了 Fiber，每一个 Element 会对应一个 Fiber Node，将 Fiber Node 链接起来的结构成为 Fiber Tree。在后续的更新过程中（setState），每次重新渲染都会重新创建Element, 但是 Fiber 不会，Fiber 只会使用对应的 Element 中的数据来更新自己必要的属性，

Fiber Tree 一个重要的特点是链表结构，将递归遍历编程循环遍历，然后配合 requestIdleCallback API, 实现任务拆分、中断与恢复。

这个链接的结构是怎么构成的呢，这就要主要到之前 Fiber Node 的节点的这几个字段：

```javascript
// 单链表树结构
{
   return: Fiber | null, // 指向父节点
   child: Fiber | null,// 指向自己的第一个子节点
   sibling: Fiber | null,// 指向自己的兄弟结构，兄弟节点的return指向同一个父节点
}
```

每一个 Fiber Node 节点与 Virtual Dom 一一对应，所有 Fiber Node 连接起来形成 Fiber tree, 是个单链表树结构，如下图所示：

![](https://pic4.zhimg.com/80/v2-a825372d761879bd1639016e6db93947_hd.jpg)

对照图来看，是不是可以知道 Fiber Node 是如何联系起来的呢，Fiber Tree 就是这样一个单链表。

**当 render 的时候有了这么一条单链表，当调用 `setState` 的时候又是如何 Diff 得到 change 的呢？**

采用的是一种叫**双缓冲技术（double buffering）**，这个时候就需要另外一颗树：WorkInProgress Tree。

在 render 的时候创建的那颗 Fiber Tree 被称作为 Current Tree，另外 setState 的时候回重新构建一颗 WorkInProgress Tree，不过不是完全的重新创建，会有一定的策略来复用  Current Tree 里的节点，这样可以节省不必要的 Node 创建。

WorkInProgress Tree 构造完毕，得到的就是新的 Fiber Tree，然后喜新厌旧（把 current 指针指向WorkInProgress Tree，丢掉旧的 Fiber Tree）就好了

这样做的好处：

- 能够复用内部对象（fiber）
- 节省内存分配、GC的时间开销
- 就算运行中有错误，也不会影响 View 上的数据

每个 Fiber上都有个`alternate`属性，也指向一个 Fiber，创建 WorkInProgress 节点时优先取`alternate`，没有的话就创建一个。

> TODO: 这里需要详解如何构造 WorkInProgress Tree 的策略。

创建 WorkInProgress Tree 的过程也是一个 Diff 的过程，Diff 完成之后会生成一个 Effect List，这个 Effect List 就是最终 Commit 阶段用来处理副作用的阶段。

### Side Effect(副作用)

我们可以将 React 中的一个组件视为一个使用 state 和 props 来计算 UI 表示的函数。其他所有活动，如改变 DOM 或调用生命周期方法，都应该被视为副作用，或者简单地说是一种效果。[文档中](https://link.juejin.im?target=https%3A%2F%2Freactjs.org%2Fdocs%2Fhooks-overview.html%23%EF%B8%8F-effect-hook) 是这样描述的：

> *您之前可能已经在 React 组件中执行数据提取，订阅或手动更改 DOM。我们将这些操作称为“副作用”（或简称为“效果”），因为它们会影响其他组件，并且在渲染过程中无法完成。*

您可以看到大多 state 和 props 更新都会导致副作用。既然使用副作用是工作（活动）的一种类型，Fiber 节点是一种方便的机制来跟踪除了更新以外的效果。每个 Fiber 节点都可以具有与之相关的副作用，它们可在 `effectTag` 字段中编码。

因此，Fiber 中的副作用基本上定义了处理更新后需要为实例完成的 [工作](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Ffacebook%2Freact%2Fblob%2Fb87aabdfe1b7461e7331abb3601d9e6bb27544bc%2Fpackages%2Fshared%2FReactSideEffectTags.js)。对于宿主组件（DOM 元素），所谓的工作包括添加，更新或删除元素。对于类组件，React可能需要更新 refs 并调用 `componentDidMount` 和 `componentDidUpdate` 生命周期方法。对于其他类型的 Fiber ，还有相对应的其他副作用。



https://juejin.im/post/5c052f95e51d4523d51c8300#heading-7

### Effects List

React 处理更新的元素要求非常迅速，为了达到这种水平的性能，它采用了一些有趣的技术。**其中之一是构建具有副作用的 Fiber 节点的线性列表，从而能够快速遍历。**遍历线性列表比树快得多，并且没有必要在没有副作用的节点上花费时间。

此列表的目标是标记具有 DOM 更新或其他相关副作用的节点。此列表是 `finishedWork` 树的子集，并使用 `nextEffect` 属性而不是 `current` 和 `workInProgress` 树中使用的 `child` 属性进行链接。



React内部有自己的优先级判断逻辑，比如动画，用户交互等任务优先级就明显要高。



> **我们需要重点理解的是，第一个 `render` 阶段的工作是可以异步执行的。**React 可以根据可用时间片来处理一个或多个 Fiber 节点，然后停下来暂存已完成的工作，并转而去处理某些事件，接着它再从它停止的地方继续执行。但有时候，它可能需要丢弃完成的工作并再次从顶部开始。由于在此阶段执行的工作不会导致任何用户可见的更改（如 DOM 更新），因此暂停行为才有了意义。**与之相反的是，后续 `commit` 阶段始终是同步的。**这是因为在此阶段执行的工作会导致用户可见的变化，例如 DOM 更新。这就是为什么 React 需要在一次单一过程中完成这些更新。

https://juejin.im/post/5c052f95e51d4523d51c8300#heading-7

## 后记

本开始想一篇文章把 Fiber 讲透的，但是写着写着发现确实太多了，想写详细，估计要写几万字，所以我这篇文章的目的仅仅是在没有涉及到源码的情况下梳理了大致 React 的工作流程，对于细节，比如如何调度异步任务、如何去做 Diff 等等细节将以小节的方式一个个的结合源码进行分析。

## 一些问题

- 如何去划分任务优先级？

- 在 reconcile 过程的 render 阶段是如何去遍历链表，如何去构建 workInProgress 的？
- 当任务被打断，如何恢复？
- 如何去收集 EffectList？
- 针对不同的组件类型如何进行更新？

## 参考

- [完全理解 React Fiber]([http://www.ayqy.net/blog/dive-into-react-fiber/#articleHeader4](http://www.ayqy.net/blog/dive-into-react-fiber/#articleHeader4))
- [Fiber](https://happy-alex.github.io/js/react/fiber/)