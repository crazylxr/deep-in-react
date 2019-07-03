```javascript
在介绍的之前需要把 一些 React 中的数据结构先介绍，不然有些链表操作不会懂！

react 的 diff 算法是从 `reconcileChildren` 开始的

export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderExpirationTime: ExpirationTime,
) {
  if (current === null) {
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderExpirationTime,
    );
  } else {
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderExpirationTime,
    );
  }
}
```



`reconcileChildren` 只是一个入口函数，如果首次渲染，current 空 null，就通过 `mountChildFibers` 创建子节点的 Fiber 实例。如果不是首次渲染，就调用 `reconcileChildFibers`去做 diff，然后得出 effect list。



接下来再看看 mountChildFibers 和 reconcileChildFibers 有什么区别：

```javascript
export const reconcileChildFibers = ChildReconciler(true);
export const mountChildFibers = ChildReconciler(false);

```

他们都是通过 `ChildReconciler` 函数来的，只是传递的参数不同而已。这个参数叫`shouldTrackSideEffects`，他的作用是判断是否要增加一些`effectTag`，主要是用来优化初次渲染的，因为初次渲染没有更新操作



