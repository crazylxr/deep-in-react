### Side Effect(副作用)

我们可以将 React 中的一个组件视为一个使用 state 和 props 来计算 UI 表示的函数。其他所有活动，如改变 DOM 或调用生命周期方法，都应该被视为副作用。[文档中](https://link.juejin.im?target=https%3A%2F%2Freactjs.org%2Fdocs%2Fhooks-overview.html%23%EF%B8%8F-effect-hook) 是这样描述的：

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