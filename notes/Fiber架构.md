## 介绍

React Fiber 是 React 核心算法的重构，这是React团队两年多研究的结果。

React Fiber的目标是增加其对动画，布局和手势等区域的适用性。它的标题功能是**增量渲染**：能够将渲染工作分割成块并将其分散到多个帧中。

其他主要功能包括在新更新进入时暂停，中止或重复工作的能力; 为不同类型的更新分配优先级的能力; 和新的并发原语。

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

一个Fiber就是一个POJO对象，代表了组件上需要做的工作。一个React Element可以对应一个或多个Fiber节点。

Fiber在update的时候，会从原来的Fiber（我们称为current）clone出一个新的Fiber（我们称为alternate）。两个Fiber diff出的变化（side effect）记录在alternate上。所以一个组件在更新时最多会有两个Fiber与其对应，在更新结束后alternate会取代之前的current的成为新的current节点。

## Fiber Tree 与 workInProgress tree

在第一次渲染之后，React 最终得到一个 Fiber 树，它反映了用于渲染 UI 的应用程序的状态。这棵树通常被称为 **current 树（当前树）**。当 React 开始处理更新时，它会构建一个所谓的 **workInProgress 树（工作过程树）**，它反映了要刷新到屏幕的未来状态。

所有工作都在 `workInProgress` 树的 Fiber 节点上执行。当 React 遍历 `current` 树时，对于每个现有 Fiber 节点，React 会创建一个构成 `workInProgress` 树的备用节点，这一节点会使用 render 方法返回的 React 元素中的数据来创建。处理完更新并完成所有相关工作后，React 将准备好一个备用树以刷新到屏幕。一旦这个 `workInProgress` 树在屏幕上呈现，它就会变成 `current` 树。

React 的核心原则之一是一致性。 React 总是一次性更新 DOM - 它不会显示部分中间结果。`workInProgress` 树充当用户不可见的「草稿」，这样 React 可以先处理所有组件，然后将其更改刷新到屏幕。

每个 Fiber 节点持有`备用`域在另一个树的对应部分的引用。来自 `current` 树中的节点会指向 `workInProgress` 树中的节点，反之亦然。

## 协调（reconciler）

## 调度

## **Side-effects**

### **Effects list**



###  参考文章

- [react fiber architectrue](https://github.com/acdlite/react-fiber-architecture)
- [React Fiber 那些事: 深入解析新的协调算法](https://segmentfault.com/a/1190000017241034)