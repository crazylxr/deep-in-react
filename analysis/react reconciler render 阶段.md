自顶向下逐节点构造workInProgress tree，不同类型的节点处理方式有差异，比如React组件，Dom节点处理方法不同。

组件处理流程

1. 如果当前节点不需要更新，则把子节点clone过来，跳到步骤5
2. 跟新当前节点的props,state,context等，调用getDerivedStateFromProps静态方法
3. 调用shouldComponentUpdate()，false的话，跳到步骤5
4. 调用组件的render方法获得新的children，标记effectTag, 进行reconcileChildren，为子节点创建fiber（创建过程会尽量复用现有fiber，子节点增删移动发生在这里，不是真实的dom节点）
5. 如果没有workInProgress.child，则工作单元结束，则进入第二阶段，diffProperties,标记tag，把effect list归并到return fibre，并把当前节点的sibling作为下一个工作单元；否则把child作为下一个工作单元
6. 如果没有剩余可用时间了，等到下一次主线程空闲时才开始下一个工作单元；否则，立即开始做
7. 当返回到root节点的时候，工作循环结束，进入commit阶段
8. completeRoot阶段，标记isWorking = true;此阶段不可打断。有三个大循环遍历effect list。

**第一个循环** commitBeforeMutationLifecycles 调用组件的getSnapshotBeforeUpdate

**第二个循环** commitAllHostEffects ，根据effectTag应用更新，有三种tag,(Placement | Update | Deletion)

**第三个循环** commitAllLifeCycles 调用生命周期，初次挂载调用componentDidMount，更新阶段调用componentDidUpdate componentWillUnmount

工作循环核心代码。

```
function workLoop(isAsync) {
  if (!isAsync) {
    // Flush all expired work.
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {
    // Flush asynchronous work until the deadline runs out of time.
    while (nextUnitOfWork !== null && !shouldYield()) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }

    if (enableProfilerTimer) {
      // If we didn't finish, pause the "actual" render timer.
      // We'll restart it when we resume work.
      pauseActualRenderTimerIfRunning();
    }
  }
}
```

这是一个典型的递归的循环写法。一来就是和传统的递归改循环写法一样，避免调用栈不断堆叠以及调用栈溢出等问题；二来在结合其他Scheduler代码的辅助变量，可以实现遍历随时终止、随时恢复的效果。