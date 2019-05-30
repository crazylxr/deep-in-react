## 例子

https://claudiopro.github.io/react-fiber-vs-stack-demo/stack.html

https://claudiopro.github.io/react-fiber-vs-stack-demo/fiber.html

## 介绍

React Fiber 是 React 核心算法的重构，这是React团队两年多研究的结果。

React Fiber的目标是增加其对动画，布局和手势等区域的适用性。它的标题功能是**增量渲染**：能够将渲染工作分割成块并将其分散到多个帧中。

其他主要功能包括在新更新进入时暂停，中止或重复工作的能力; 为不同类型的更新分配优先级的能力; 和新的并发原语。

**解决的问题：用户的鼠标，键盘事件没办法相应的卡顿**



因为JavaScript单线程的特点，每个同步任务不能耗时太长，不然就会让程序不会对其他输入作出相应，React的更新过程就是犯了这个禁忌，而React Fiber就是要改变现状

## 目标

- 暂停工作，稍后再回来。
- 为不同类型的工作分配优先权。
- 重用以前完成的工作。
- 如果不再需要，则中止工作。

为了做到这一点，我们首先需要一种方法将工作分解为单元。从某种意义上说，这就是 Fiber，Fiber 代表一种**工作单元**。

## 概述

首先整个分成两个阶段，第一个阶段是**reconciliation**，第二个阶段是 **commit**。

**reconciliation** 阶段的目标就是 Fiber 的 **update**，然后产出的是 **effect list**（可以想象成将老的 View 更新到新的状态所需要做的 DOM 操作的列表）。这一个阶段是没有副作用的，因此这个过程可以被打断，然后恢复执行。

**commit** 阶段是 Reconciliation 产生的 effect list 只有在 commit 之后才会生效，也就是真正应用到 DOM 中。这一阶段往往不会执行太长时间，因此是同步的，这样也避免了组件内视图层结构和 DOM 不一致。

## Fiber

通常称为纤程，跟 go 的 coroutine(协程)类似

大家应该都清楚进程（Process）和线程（Thread）的概念，在计算机科学中还有一个概念叫做Fiber，英文含义就是“纤维”，意指比Thread更细的线，也就是比线程(Thread)控制得更精密的并发处理机制。

上面说的Fiber和React Fiber不是相同的概念，但是，我相信，React团队把这个功能命名为Fiber，含义也是更加紧密的处理机制，比Thread更细。





Fiber 对象出现的原因是之前的 vdom 节点没有记录上下文的信息，一旦中断无法恢复。

一个Fiber就是一个POJO对象，代表了组件上需要做的工作。一个React Element可以对应一个或多个Fiber节点。

Fiber在update的时候，会从原来的Fiber（我们称为current）clone出一个新的Fiber（我们称为alternate）。两个Fiber diff出的变化（side effect）记录在alternate上。所以一个组件在更新时最多会有两个Fiber与其对应，在更新结束后alternate会取代之前的current的成为新的current节点。



低优先级的代码会被打断，重新执行

## Fiber Tree 与 workInProgress tree

在第一次渲染之后，React 最终得到一个 Fiber 树，它反映了用于渲染 UI 的应用程序的状态。这棵树通常被称为 **current 树（当前树）**。当 React 开始处理更新时，它会构建一个所谓的 **workInProgress 树（工作过程树）**，它反映了要刷新到屏幕的未来状态。

所有工作都在 `workInProgress` 树的 Fiber 节点上执行。当 React 遍历 `current` 树时，对于每个现有 Fiber 节点，React 会创建一个构成 `workInProgress` 树的备用节点，这一节点会使用 render 方法返回的 React 元素中的数据来创建。处理完更新并完成所有相关工作后，React 将准备好一个备用树以刷新到屏幕。一旦这个 `workInProgress` 树在屏幕上呈现，它就会变成 `current` 树。

React 的核心原则之一是一致性。 React 总是一次性更新 DOM - 它不会显示部分中间结果。`workInProgress` 树充当用户不可见的「草稿」，这样 React 可以先处理所有组件，然后将其更改刷新到屏幕。

每个 Fiber 节点持有`备用`域在另一个树的对应部分的引用。来自 `current` 树中的节点会指向 `workInProgress` 树中的节点，反之亦然。



 workInProgress tree是reconcile过程中从fiber tree建立的当前进度快照，用于断点恢复

双缓冲技术（double buffering），就像[redux里的`nextListeners`](http://www.ayqy.net/blog/redux源码解读/#articleHeader7)，以fiber tree为主，workInProgress tree为辅

双缓冲具体指的是workInProgress tree构造完毕，得到的就是新的fiber tree，然后喜新厌旧（把current指针指向workInProgress tree，丢掉旧的fiber tree）就好了

这样做的好处：

- 能够复用内部对象（fiber）
- 节省内存分配、GC的时间开销

> 问题，为什么会节省内存分配，GC 的时间开销

## 更新

两个阶段，reconciler和commit

## 协调（reconciler）

以fiber tree为蓝本，把每个fiber作为一个工作单元，自顶向下逐节点构造*workInProgress tree*（构建中的新fiber tree）

具体过程如下（以组件节点为例）：

1. 如果当前节点不需要更新，直接把子节点clone过来，跳到5；要更新的话打个tag
2. 更新当前节点状态（`props, state, context`等）
3. 调用`shouldComponentUpdate()`，`false`的话，跳到5
4. 调用`render()`获得新的子节点，并为子节点创建fiber（创建过程会尽量复用现有fiber，子节点增删也发生在这里）
5. 如果没有产生child fiber，该工作单元结束，把effect list归并到return，并把当前节点的sibling作为下一个工作单元；否则把child作为下一个工作单元
6. 如果没有剩余可用时间了，等到下一次主线程空闲时才开始下一个工作单元；否则，立即开始做
7. 如果没有下一个工作单元了（回到了workInProgress tree的根节点），第1阶段结束，进入pendingCommit状态



> 疑问1 ：如果判断节点是否需要更新



来自：完全理解 [React Fiber]([http://www.ayqy.net/blog/dive-into-react-fiber/#articleHeader4](http://www.ayqy.net/blog/dive-into-react-fiber/#articleHeader4))

## 调度

调度（scheduling）：是一个决定什么时候该做某个任务的过程。

## **Side-effects**



### **Effects list**

每个workInProgress tree节点上都有一个effect list
用来存放diff结果
当前节点更新完毕会向上merge effect list（queue收集diff结果）

## 优缺点分析

### 缺点

比如说，task按照优先级之后，可能低优先级的任务永远不会执行，称之为starvation；

比如说，task有可能被打断，需要重新执行，那么某些依赖生命周期实现的业务逻辑可能会受到影响。

### 优点

比如说，增强了某些领域的支持，如动画、布局和手势；

比如说，在复杂页面，对用户的反馈会更及时，应用的用户体验会变好，简单页面看不到明显的差异；

比如说，api基本上没有变化，对现有项目很友好。



## 一些简单的 react

https://github.com/y805939188/simple-react

https://github.com/Luminqi/learn-react

###  参考文章

- [react fiber architectrue](https://github.com/acdlite/react-fiber-architecture)
- [React Fiber 那些事: 深入解析新的协调算法](https://segmentfault.com/a/1190000017241034)
- [协程](https://www.liaoxuefeng.com/wiki/897692888725344/923057403198272)