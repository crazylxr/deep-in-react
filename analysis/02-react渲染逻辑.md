核心知识点

- 是否有 deadline
- 循环渲染 Root 条件
- 超过时间片的处理

```javascript
function performAsyncWork() {
  try {
    if (!shouldYieldToRenderer()) { // 其实理论上应该就是 shouldYield 的返回值
      // The callback timed out. That means at least one update has expired.
      // Iterate through the root schedule. If they contain expired work, set
      // the next render expiration time to the current time. This has the effect
      // of flushing all expired work in a single batch, instead of flushing each
      // level one at a time.
      // 回调超时。 这意味着至少有一个更新已过期。 迭代 root schedule。 
      if (firstScheduledRoot !== null) {
        recomputeCurrentRendererTime();
        let root: FiberRoot = firstScheduledRoot;
        
        // 这个循环是找到所有过期的 Root 节点，然后设置为当前时间，这样就可以在一个批次中清楚所有过期的 work。
        do {
          // 如果它们包含过期的工作，将下一个渲染过期时间设置为当前时间。
          didExpireAtExpirationTime(root, currentRendererTime);
          // The root schedule is circular, so this is never null.
          root = (root.nextScheduledRoot: any);
        } while (root !== firstScheduledRoot);
      }
    }
    performWork(NoWork, true);
  } finally {
    didYield = false;
  }
}

```

```javascript
function performSyncWork() {
  performWork(Sync, false);
}


```

## performWork

执行工作的函数。

performWork的作用就是“刷新”待更新队列，执行待更新的事务：

既然要执行 work，那么首先得需要知道下一步要执行哪一个 work。

`findHigestPriorityRoot` 函数就是用来找到最高优先级的 work，然后进行执行。

```javascript

function performWork(minExpirationTime: ExpirationTime, isYieldy: boolean) {
  // Keep working on roots until there's no more work, or until there's a higher
  // priority event.
  // 继续 work，直到没有更多的 work，或直到有更高优先级的事件。
  // 修改 nextFlushedRoot 、nextFlushedExpirationTime 两个全局变量
  findHighestPriorityRoot();

  // isYieldy 在同步的情况下是false，而在异步情况下是true
  if (isYieldy) {
    recomputeCurrentRendererTime();
    currentSchedulerTime = currentRendererTime;
     
    // 这个跟我们的主流程没有关系，直接忽略掉
    if (enableUserTimingAPI) {
      const didExpire = nextFlushedExpirationTime > currentRendererTime;
      const timeout = expirationTimeToMs(nextFlushedExpirationTime);
      stopRequestCallbackTimer(didExpire, timeout);
    }

    while (
      nextFlushedRoot !== null &&
      nextFlushedExpirationTime !== NoWork &&
      minExpirationTime <= nextFlushedExpirationTime &&
      !(didYield && currentRendererTime > nextFlushedExpirationTime) // 这个判断代表任务过期
      // 在帧未到期 或者 当前渲染时间大于等于nextFlushedExpirationTime时才执行 performWorkOnRoot, 并将currentRendererTime >= nextFlushedExpirationTime作为第三个参数传入，

    ) {
      performWorkOnRoot(
        nextFlushedRoot,
        nextFlushedExpirationTime,
        currentRendererTime > nextFlushedExpirationTime,
      );
      findHighestPriorityRoot();
      recomputeCurrentRendererTime();
      currentSchedulerTime = currentRendererTime;
    }
  } else {
    // 同步的时候 minExpirationTime 是 SYNC，是最大的，所以nextFlushedExpirationTime只能是 SYNC 才行
    while (
      nextFlushedRoot !== null &&
      nextFlushedExpirationTime !== NoWork &&
      minExpirationTime <= nextFlushedExpirationTime
    ) {
      // 会执行掉这个work ，并把他设置为空，不然下次循环还在
      performWorkOnRoot(nextFlushedRoot, nextFlushedExpirationTime, false);
      findHighestPriorityRoot();
    }
  }

  // We're done flushing work. Either we ran out of time in this callback,
  // or there's no more work left with sufficient priority.

  // If we're inside a callback, set this to false since we just completed it.
  if (isYieldy) {
    callbackExpirationTime = NoWork;
    callbackID = null;
  }
  // If there's work left over, schedule a new callback.
  if (nextFlushedExpirationTime !== NoWork) {
    scheduleCallbackWithExpirationTime(
      ((nextFlushedRoot: any): FiberRoot),
      nextFlushedExpirationTime,
    );
  }

  // Clean-up.
  finishRendering();
}
```

## findHighestPriorityRoot

这个方法是从 RootSchedule 里面找到最高优先级的节点和 expirationTime

在这些 Root 节点里面分为两种，一种是需要更新的节点，一种是不需要更新的节点，对于不需要更新的节点，应该从 schedule 里面将其删除。

而对于需要更新的节点，把他的 expirationTime 与当前最高的 expirationTime 作比较，比他大，就比它设置为最高优先级的，这也是我们熟悉的从链表中找最大的值的方法。

```javascript
function findHighestPriorityRoot() {
  let highestPriorityWork = NoWork;
  let highestPriorityRoot = null;
  // 最有一个 root 不为空 ，意味着有要更新的节点
  if (lastScheduledRoot !== null) {
    let previousScheduledRoot = lastScheduledRoot;
    let root = firstScheduledRoot;
    while (root !== null) {
      const remainingExpirationTime = root.expirationTime;
      if (remainingExpirationTime === NoWork) {
        // 因为 expirationTime 等于 NoWork ，所以在这种情况下的节点都应该从 schedule 里面删除
        
        if (root === root.nextScheduledRoot) { // 代表就只有这一个节点
          // This is the only root in the list.
          root.nextScheduledRoot = null;
          firstScheduledRoot = lastScheduledRoot = null;
          break;
        } else if (root === firstScheduledRoot) {
          // This is the first root in the list.
          const next = root.nextScheduledRoot;
          firstScheduledRoot = next;
          lastScheduledRoot.nextScheduledRoot = next;
          root.nextScheduledRoot = null;
        } else if (root === lastScheduledRoot) {
          // This is the last root in the list.
          lastScheduledRoot = previousScheduledRoot;
          lastScheduledRoot.nextScheduledRoot = firstScheduledRoot;
          root.nextScheduledRoot = null;
          break;
        } else {
          previousScheduledRoot.nextScheduledRoot = root.nextScheduledRoot;
          root.nextScheduledRoot = null;
        }
        root = previousScheduledRoot.nextScheduledRoot;
      } else {
        if (remainingExpirationTime > highestPriorityWork) {
          // Update the priority, if it's higher
          // 找到整个链表中最高优先级的节点和expirationTime;
          highestPriorityWork = remainingExpirationTime;
          highestPriorityRoot = root;
        }
        if (root === lastScheduledRoot) {
          break;
        }
        if (highestPriorityWork === Sync) {
          // Sync is highest priority by definition so
          // we can stop searching.
          break;
        }
        previousScheduledRoot = root;
        root = root.nextScheduledRoot;
      }
    }
  }

  nextFlushedRoot = highestPriorityRoot;
  nextFlushedExpirationTime = highestPriorityWork;
}
```

# performWorkOnRoot

执行 root 的函数。

```javascript
function performWorkOnRoot(
  root: FiberRoot,
  expirationTime: ExpirationTime,
  isYieldy: boolean,
) {
  isRendering = true;

  // Check if this is async work or sync/expired work.
  // 检查这是否是异步工作或同步/过期工作。同步和过期任务执行逻辑
  if (!isYieldy) {
    // Flush work without yielding.
    // TODO: Non-yieldy work does not necessarily imply expired work. A renderer
    // may want to perform some work without yielding, but also without
    // requiring the root to complete (by triggering placeholders).

    let finishedWork = root.finishedWork;
    // 一般来说进来都是 null 的，不过有上次被中断的情况，那么 finnishWork 就是有值的，所以需要判断一下。这个 finishwork 是在进入提交阶段之前的 work
    if (finishedWork !== null) {
      // This root is already complete. We can commit it.
      completeRoot(root, finishedWork, expirationTime);
    } else {
      root.finishedWork = null;
      // If this root previously suspended, clear its existing timeout, since
      // we're about to try rendering again.
      // 如果该 root 提前暂停，请清除其现有超时，因为我们将要再次尝试渲染。
      const timeoutHandle = root.timeoutHandle;
      if (timeoutHandle !== noTimeout) {
        root.timeoutHandle = noTimeout;
        // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
        cancelTimeout(timeoutHandle);
      }
      // renderRoot 执行完之后会赋值 finishedWork，这个work 是 rootWorkInProgress，不是 root
      renderRoot(root, isYieldy);
      finishedWork = root.finishedWork;
      if (finishedWork !== null) {
        // We've completed the root. Commit it.
        completeRoot(root, finishedWork, expirationTime);
      }
    }
  } else {
    // Flush async work.
    let finishedWork = root.finishedWork;
    if (finishedWork !== null) {
      // This root is already complete. We can commit it.
      completeRoot(root, finishedWork, expirationTime);
    } else {
      root.finishedWork = null;
      // If this root previously suspended, clear its existing timeout, since
      // we're about to try rendering again.
      const timeoutHandle = root.timeoutHandle;
      if (timeoutHandle !== noTimeout) {
        root.timeoutHandle = noTimeout;
        // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
        cancelTimeout(timeoutHandle);
      }
      renderRoot(root, isYieldy);
      finishedWork = root.finishedWork;
      if (finishedWork !== null) {
        // We've completed the root. Check the if we should yield one more time
        // before committing.
        if (!shouldYieldToRenderer()) {
          // Still time left. Commit the root.
          completeRoot(root, finishedWork, expirationTime);
        } else {
          // There's no time left. Mark this root as complete. We'll come
          // back and commit it later.
          root.finishedWork = finishedWork;
        }
      }
    }
  }

  isRendering = false;
}
```

