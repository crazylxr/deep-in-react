**核心知识点**

- 加入到 root 调度队列
- 判断是否批量更新
- 根据 expirationTime 判断调度类型



不懂什么是异步任务，处于ConcurrentMode下的子树的渲染就是异步



```javascript
// requestWork is called by the scheduler whenever a root receives an update.
// It's up to the renderer to call renderRoot at some point in the future.

/**
 * @param {  roo节点的  rootExpirationTime } expirationTime
 */
function requestWork(root: FiberRoot, expirationTime: ExpirationTime) {
  addRootToSchedule(root, expirationTime);
  if (isRendering) { // 为 true 表示已经在调度了
    // Prevent reentrancy. Remaining work will be scheduled at the end of
    // the currently rendering batch.
    return;
  }

  // 批处理相关的
  if (isBatchingUpdates) {
    // Flush work at the end of the batch.
    if (isUnbatchingUpdates) {
      // ...unless we're inside unbatchedUpdates, in which case we should
      // flush it now.
      nextFlushedRoot = root;
      nextFlushedExpirationTime = Sync;
      performWorkOnRoot(root, Sync, false);
    }
    return;
  }

  // TODO: Get rid of Sync and use current time?
  if (expirationTime === Sync) {
    performSyncWork();
  } else { 
    // 异步任务调用的方法
    scheduleCallbackWithExpirationTime(root, expirationTime);
  }
}
```



```javascript
function addRootToSchedule(root: FiberRoot, expirationTime: ExpirationTime) {
  // Add the root to the schedule.
  // Check if this root is already part of the schedule.
  // 检查是否 root 已经在 schedule 中了
  if (root.nextScheduledRoot === null) {  // 表示现在没有任务在计划中
    // This root is not already scheduled. Add it.
    // 如果没有在计划中，就添加它
    root.expirationTime = expirationTime;
    if (lastScheduledRoot === null) {
      firstScheduledRoot = lastScheduledRoot = root;
      root.nextScheduledRoot = root;
    } else {
      // 有的话，就加入到链表的尾部
      lastScheduledRoot.nextScheduledRoot = root;
      lastScheduledRoot = root;
      lastScheduledRoot.nextScheduledRoot = firstScheduledRoot;
    }
  } else {
    // This root is already scheduled, but its priority may have increased.
    // 此 root 进入过调度，但其优先级可能已增加。
    const remainingExpirationTime = root.expirationTime;
    if (expirationTime > remainingExpirationTime) {
      // Update the priority.
      root.expirationTime = expirationTime;
    }
  }
}
```

