

- 找到更新对应的 FiberRoot 节点
- 如果符合条件重置 stack
- 如果符合条件就请求工作调度

每一次进入调度队列的都是 FiberRoot，不是 setState 的节点

```javascript

function scheduleWork(fiber: Fiber, expirationTime: ExpirationTime) {
  // 在 render 的时候传的就是FiberRoot 节点，但是 setState 的时候不是，所以要去找到 FiberRoot
  const root = scheduleWorkToRoot(fiber, expirationTime);
  if (root === null) {
    return;
  }

  // 高优先级的任务打断了老的任务。
  // isWorking 表示现在是否有任务正 在执行
  // nextRenderExpirationTime 表示下一次渲染节点的 ExpirationTime
  // 那么合起来判断的就是是否是 working，然后有下一次渲染，并且下一次的 ExpirationTime 小于 expirationTime，那么就代表目前的任务的优先级比较高，所以就暂停下一个的，直接 resetStack
  if (
    !isWorking &&
    nextRenderExpirationTime !== NoWork &&
    expirationTime > nextRenderExpirationTime
  ) {
    // This is an interruption. (Used for performance tracking.)
    // 没有用，只是用来做记录的
    interruptedBy = fiber;
    resetStack();
  }
  markPendingPriorityLevel(root, expirationTime);
  if (
    // If we're in the render phase, we don't need to schedule this root
    // for an update, because we'll do it before we exit...
    // 如果我们处于 render 阶段，我们不需要对 root 进行更新，因为会在退出之前执行这个操作
    // isWorking 代表两个阶段，而 isCommiting 是第二个阶段，同步到真实 dom 的阶段，不可打断
    !isWorking ||
    isCommitting ||
    // ...unless this is a different root than the one we're rendering.
    // 如果只有一个 root ，这个条件基本不会满足，因为只有一个 root 的情况，nextRoot 永远等于 root
    nextRoot !== root
  ) {
    const rootExpirationTime = root.expirationTime;
    requestWork(root, rootExpirationTime);
  }
  
  // 防止无限
  if (nestedUpdateCount > NESTED_UPDATE_LIMIT) {
    // Reset this back to zero so subsequent updates don't throw.
    nestedUpdateCount = 0;
    invariant(
      false,
      'Maximum update depth exceeded. This can happen when a ' +
        'component repeatedly calls setState inside ' +
        'componentWillUpdate or componentDidUpdate. React limits ' +
        'the number of nested updates to prevent infinite loops.',
    );
  }
}
```



```javascript

/**
 * @description 根据传进来的 Fiber 向上找到 根节点
 */
function scheduleWorkToRoot(fiber: Fiber, expirationTime): FiberRoot | null {
  // 做了一个记录，相当于什么都没做
  recordScheduleUpdate();

  // Update the source fiber's expiration time
  // 设置为优先级更高的 expirationTime
  if (fiber.expirationTime < expirationTime) {
    fiber.expirationTime = expirationTime;
  }
  let alternate = fiber.alternate;
  if (alternate !== null && alternate.expirationTime < expirationTime) {
    alternate.expirationTime = expirationTime;
  }
  // Walk the parent path to the root and update the child expiration time.
  // 获取到父节点
  let node = fiber.return;
  let root = null; //  其实是 FiberRoot

  // 如果 node 等于 null 代表就是 RootFiber
  if (node === null && fiber.tag === HostRoot) {
    root = fiber.stateNode; // RootFiber 的 stateNode 就是 FiberRoot
  } else { 
    // 从下往上设置 childExpirationTime，直到 node 为 null 的时候就是 root
    while (node !== null) {
      alternate = node.alternate;
      // node.childExpirationTime 是所有子节点里面优先级最高的 expirationTime
      if (node.childExpirationTime < expirationTime) {
        node.childExpirationTime = expirationTime;
        if (
          alternate !== null &&
          alternate.childExpirationTime < expirationTime
        ) {
          alternate.childExpirationTime = expirationTime;
        }
      } else if (
        alternate !== null &&
        alternate.childExpirationTime < expirationTime
      ) {
        alternate.childExpirationTime = expirationTime;
      }
      if (node.return === null && node.tag === HostRoot) {
        root = node.stateNode;
        break;
      }
      node = node.return;
    }
  }

  // 跟踪应用更新，不用深入了解
  if (enableSchedulerTracing) {
    if (root !== null) {
      const interactions = __interactionsRef.current;
      if (interactions.size > 0) {
        const pendingInteractionMap = root.pendingInteractionMap;
        const pendingInteractions = pendingInteractionMap.get(expirationTime);
        if (pendingInteractions != null) {
          interactions.forEach(interaction => {
            if (!pendingInteractions.has(interaction)) {
              // Update the pending async work count for previously unscheduled interaction.
              interaction.__count++;
            }

            pendingInteractions.add(interaction);
          });
        } else {
          pendingInteractionMap.set(expirationTime, new Set(interactions));

          // Update the pending async work count for the current interactions.
          interactions.forEach(interaction => {
            interaction.__count++;
          });
        }

        const subscriber = __subscriberRef.current;
        if (subscriber !== null) {
          const threadID = computeThreadID(
            expirationTime,
            root.interactionThreadID,
          );
          subscriber.onWorkScheduled(interactions, threadID);
        }
      }
    }
  }
  return root;
}
```

找到根节点，同时更新所有节点的 childExpirationTime

```javascript
function resetStack() {
  // nextUnitOfWork 用于记录render阶段Fiber树遍历过程中下一个需要执行的节点。
  if (nextUnitOfWork !== null) { // 代表之前更新的是异步任务，并且执行到一半由于时间片不够，吧执行权给浏览器
    let interruptedWork = nextUnitOfWork.return;
    
    // 这样理论上也是找到根节点
    while (interruptedWork !== null) {
      // 有可能已经有节点已经更新了，需要重置回未更新的状态
      unwindInterruptedWork(interruptedWork);
      interruptedWork = interruptedWork.return;
    }
  }

  nextRoot = null;
  nextRenderExpirationTime = NoWork;
  nextLatestAbsoluteTimeoutMs = -1;
  nextRenderDidError = false;
  nextUnitOfWork = null;
}
```





