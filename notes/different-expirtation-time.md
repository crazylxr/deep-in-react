异步任务优先级比较低，可能会被打断，为了防止一直被打断，一直不能执行，react 设置了一个 expirationTime，在这个时间之前，你都可以打断我，一旦过了这个时间，这个任务就不能被打断。 

**种类**

- Sync 模式
- 异步模式
- 指定 context

```javascript
/*** react-reconciler/src/ReactFiberScheduler.js ***/

function computeExpirationForFiber(currentTime: ExpirationTime, fiber: Fiber) {
  let expirationTime;
  
  // 不等于 NoWork 表示，就是 Sync，那么 expirationTime 就是 MAX_SIGNED_31_BIT_INT。外部强制指定了 expirationContext
  if (expirationContext !== NoWork) {
    // An explicit expiration context was set;
    expirationTime = expirationContext;
  } else if (isWorking) {
    if (isCommitting) {
      // Updates that occur during the commit phase should have sync priority
      // by default.
      expirationTime = Sync;
    } else {
      // Updates during the render phase should expire at the same time as
      // the work that is being rendered.
      expirationTime = nextRenderExpirationTime;
    }
  } else {
    // No explicit expiration context was set, and we're not currently
    // performing work. Calculate a new expiration time.
    
    // 没有指定 expirationContext,会创建一个新的 expirationTime
    // 判断 mode 是否处于 concurrentMode 
    if (fiber.mode & ConcurrentMode) {
      if (isBatchingInteractiveUpdates) {
        // This is an interactive update
        expirationTime = computeInteractiveExpiration(currentTime);
      } else {
        // This is an async update
        expirationTime = computeAsyncExpiration(currentTime);
      }
      // If we're in the middle of rendering a tree, do not update at the same
      // expiration time that is already rendering.
      // 如果正在渲染一个树，就不渲染相同到期时间进行更新。
      if (nextRoot !== null && expirationTime === nextRenderExpirationTime) {
        expirationTime -= 1;
      }
    } else {
      // This is a sync update
      expirationTime = Sync;
    }
  }
  if (isBatchingInteractiveUpdates) {
    // This is an interactive update. Keep track of the lowest pending
    // interactive expiration time. This allows us to synchronously flush
    // all interactive updates when needed.
    if (
      lowestPriorityPendingInteractiveExpirationTime === NoWork ||
      expirationTime < lowestPriorityPendingInteractiveExpirationTime
    ) {
      lowestPriorityPendingInteractiveExpirationTime = expirationTime;
    }
  }
  return expirationTime;
}
```
