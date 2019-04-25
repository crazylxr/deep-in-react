在 React 中创建更新的方式常用的有三种：

- ReactDOM.render
- setState
- forceUpdate

首先我们调用的就是  `ReactDOM.render` 方法，接下来就从这个方法开始解析源码：

> 我在贴源码分析的时候是删除了开发的时候的代码，也就是 `if(__DEV__)`里面的代码。

我们找到在 ReactDOM.js 找到 ReactDOM 对象，可以看到 ReactDOM 还是有挺多方法属性的，但是大部分 API 都是没有在文档里面体现，所以我们也不会用，我们直接进入到 ReactDOM 的 render 方法分析，这是 React 渲染的入口：

```javascript
/*** react-dom/src/client/ReactDOM.js ***/

const ReactDOM: Object = {
    // element: 传进来的 App 组件
    // container: 挂载的 DOM 节点
    // callback: 回调函数
    render(
    element: React$Element<any>,
    container: DOMContainer,
    callback: ?Function,
  ) {
    return legacyRenderSubtreeIntoContainer(
      null,
      element,
      container,
      false,
      callback,
    );
  },
}
```

render 方法直接返回的是 `legacyRenderSubtreeIntoContainer` 方法的返回值。

```javascript
/*** react-dom/src/client/ReactDOM.js ***/

// parentComponent 初次是 null
// children 就是 ReactDOM.render 传进来的第一个参数，就是根对象
// container 是需要挂载的 DOM 节点
// forceHydrate 我们讨论的浏览器端的，固定是 false，如果是服务端的就是 true
// callback 就是 调用 ReactDOM.render 传进来的第三个参数，callback
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  forceHydrate: boolean,
  callback: ?Function,
) {
      
  // 首次 render 的时候，肯定 container 下面是没有 _reactRootContainer 的，所以是 null
  let root: Root = (container._reactRootContainer: any);
  // 首次渲染 root 为 null，执行 if
  if (!root) {
    // Initial mount
    //  初始化渲染，没有 root，那么就通过 legacyCreateRootFromDOMContainer 创建一个 root
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    
    // 如果有 callback 就执行，做了一个简单的封装，把节点信息传递给回调函数
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(root._internalRoot);
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched.
    // 首次渲染不应该批量更新，这个方法暂时可以理解成会马上执行这个参数传递的函数就行，其实里面只是去设置了一个全局变量
    unbatchedUpdates(() => {
      if (parentComponent != null) {
        root.legacy_renderSubtreeIntoContainer(
          parentComponent,
          children,
          callback,
        );
      } else {
        root.render(children, callback);
      }
    });
  } else { 
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(root._internalRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    if (parentComponent != null) {
      root.legacy_renderSubtreeIntoContainer(
        parentComponent,
        children,
        callback,
      );
    } else {
      root.render(children, callback);
    }
  }
  return getPublicRootInstance(root._internalRoot);
}
```

在这个函数总结一下就是，没有 `_reactRootContainer` 就创建一个 **root**，然后根据是否 parentComponent 为 null 来调用 root 的 legacy_renderSubtreeIntoContainer 方法或者是 render 方法，然后返回 getPublicRootInstance 的返回值。 

然后看 root 创建的对象是什么，所以现在我们看创建 root 的函数 `legacyCreateRootFromDOMContainer`

```javascript
/*** react-dom/src/client/ReactDOM.js ***/

/**
 *@param container 是需要挂载的 DOM 节点
 *@param forceHydrate 我们讨论的浏览器端的，固定是 false，如果是服务端的就是 true
 */
function legacyCreateRootFromDOMContainer(
  container: DOMContainer,
  forceHydrate: boolean,
): Root {
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  // First clear any existing content.
  // 首次清楚所有的子节点
  if (!shouldHydrate) {
    let warned = false;
    let rootSibling;
    while ((rootSibling = container.lastChild)) {
      container.removeChild(rootSibling);
    }
  }
  // Legacy roots are not async by default.
  // 默认是是同步的
  const isConcurrent = false;
  return new ReactRoot(container, isConcurrent, shouldHydrate);
}
```

legacyCreateRootFromDOMContainer 这个函数在浏览器端的作用就是清楚挂载节点的所有子节点，然后根据 container(需要挂载的 DOM 节点)返回一个 ReactRoot 实例。

那我们再看看 ReactRoot 的构造函数，看看它的数据结构是怎么样的。

```javascript
/*** react-dom/src/client/ReactDOM.js ***/

function ReactRoot(
  container: DOMContainer,
  isConcurrent: boolean,
  hydrate: boolean,
) {
  const root = createContainer(container, isConcurrent, hydrate);
  this._internalRoot = root;
}
```

ReactRoot 很简单，就是将 createContainer 返回值赋值给 `_internalRoot`， createContainer 函数返回值其实是一个 FiberRoot，FiberRoot 是怎么样的，暂时不讨论，后面再详细介绍。

好，现在我们再回到 legacyRenderSubtreeIntoContainer 函数，当创建了 root 之后，然后会进行渲染，就是 root.render，接下来我们学习一下这个。

在前面的分析中，得知 legacyRenderSubtreeIntoContainer 中的 root 其实就是 legacyCreateRootFromDOMContainer 返回的值，而 legacyCreateRootFromDOMContainer 返回的就是 ReactRoot 的实例，所以 `root.render` 就是调用的 ReactRoot 实例上的 render 方法。

```javascript
/*** react-dom/src/client/ReactDOM.js ***/

function ReactRoot(
  container: DOMContainer,
  isConcurrent: boolean,
  hydrate: boolean,
) {
 
  const root = createContainer(container, isConcurrent, hydrate);
  this._internalRoot = root;
}
ReactRoot.prototype.render = function(
  children: ReactNodeList,
  callback: ?() => mixed,
): Work {
  // 这个 root 为构造方法里面通过 createContainer 创建的 FiberRoot
  const root = this._internalRoot;
  // Work 相关的是用来执行 callback 的，不是太重要，先忽略
  const work = new ReactWork();
  callback = callback === undefined ? null : callback;

  if (callback !== null) {
    work.then(callback);
  }
  // 调用 updateContainer 来进行更新
  updateContainer(children, root, null, work._onCommit);
  return work;
};
```

那么这个 updateContainer 函数就是一个很重要的函数了，现在我们来研究一下。

```javascript
/***  react-reconciler/ReactFiberReconciler.js  ***/

/**
 *
 * @param { ReactDOM.render 传进来的第一个参数，就是根对象 } element
 * @param { FiberRoot 实例 } container
 * @param {*} parentComponent
 * @param {*} callback
 */

export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {
  // 这个 current 是一个 Fiber
  const current = container.current;
  // 暂时可以理解为当前时间
  const currentTime = requestCurrentTime();
  // 过期时间的计算，非常重要
  const expirationTime = computeExpirationForFiber(currentTime, current);
  return updateContainerAtExpirationTime(
    element,
    container,
    parentComponent,
    expirationTime,
    callback,
  );
}
```

得到 RootFiber ，然后得到 currentTime，计算 expirationTime，最后通过 expirationTime 创建更新，接下来再来看如果创建更新的。

```javascript
/***  react-reconciler/ReactFiberReconciler.js  ***/

/**
 *
 * @param { ReactDOM.render 传进来的第一个参数，就是根对 } element
 * @param { FiberRoot 实例  } container
 * @param {*} parentComponent
 * @param { 过期时间 } expirationTime
 * @param { 封装过后的 callback } callback
 */
export function updateContainerAtExpirationTime(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {
  // TODO: If this is a nested container, this won't be the root.
  // 获取 root 节点对应的 Fiber，FiberRoot.current -> RootFiber ,RootFiber.stateNode -> FiberRoot
  const current = container.current;

  // 初次渲染的时候 parentComponent 为空，所以获取不到 context，结果其实是一个空对象({})
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  // 调度更新
  return scheduleRootUpdate(current, element, expirationTime, callback);
}
```

```javascript
/***  react-reconciler/ReactFiberReconciler.js  ***/

/**
 *
 * @param { 就是当前节点对应的 Fiber } current
 * @param { ReactDOM.render 传进来的第一个参数，就是根对象 } element
 * @param { 过期时间 } expirationTime
 * @param { 封装过的 callback } callback
 */
function scheduleRootUpdate(
  current: Fiber,
  element: ReactNodeList,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {

  // 根据 expirationTime 创建一个 Update 对象，后面会介绍
  const update = createUpdate(expirationTime);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  // 如果是 render 那么 payload 回事传进来的根节点，如果是 setState 调用scheduleRootUpdate，那么 payload 就是 setState 的第一个参数
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    warningWithoutStack(
      typeof callback === 'function',
      'render(...): Expected the last optional `callback` argument to be a ' +
        'function. Instead received: %s.',
      callback,
    );
    update.callback = callback;
  }

  // 不重要，暂时不看
  flushPassiveEffects();
  // 创建更新队列，将前面的 Update 对象添加到队列里。这个函数在介绍完 Fiber 架构之后再来解释，不然看不懂
  enqueueUpdate(current, update);
  // 任务调度，进行更新过程;因为在现在的 React 里有很多的任务，所以多个任务之间的协调是非常重要的。
  scheduleWork(current, expirationTime);

  return expirationTime;
}
```

#### scheduleWork

- 找到更新对应的 FiberRoot 节点
- 如果符合条件重置 stack
- 如果符合条件就请求工作调度

每一次进入调度队列的都是 FiberRoot，不是 setState 的节点

```javascript
/*** react-reconciler/src/ReactFiberScheduler.js ***/
/**
 * @param { 就是当前节点对应的 Fiber } fiber
 * @param { 计算出来的过期时间 } expirationTime
 */
function scheduleWork(fiber: Fiber, expirationTime: ExpirationTime) {
  // 在 render 的时候传的就是FiberRoot 节点，但是 setState 的时候不是，所以要去找到 FiberRoot
  const root = scheduleWorkToRoot(fiber, expirationTime);
  if (root === null) {
    return;
  }

  // isWorking 代表是否正在工作，在开始renderRoot和commitRoot的时候会设置为 true，也就是在render和commit两个阶段都会为true
  // nextRenderExpirationTime 在是新的 renderRoot 的时候会被设置为当前任务的 expirationTime，而且一旦他被赋值，只有当下次任务是 NoWork 的时候他才会被再次设置为 NoWork，当然最开始也是 NoWork
  // 目前没有任何任务在执行，并且之前有执行过任务，同时当前的任务比之前执行的任务过期时间要早（也就是优先级要高）
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
  // 这个方法会记录当前的 expirationTime 到 pendingTime，让 expirationTime 处于 earliestPendingTime 和 latestPendingTime 之间
  // 并且会设置 root.nextExpirationTimeToWorkOn 和 root.expirationTime = expirationTime 分别是：
  // 最早的 pendingTime 或者 pingedTime，如果都没有则是 lastestSuspendTime
  // suspendedTime 和 nextExpirationTimeToWorkOn 中较早的一个
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
/*** react-reconciler/src/ReactFiberScheduler.js ***/

/**
 * @description 根据传进来的 Fiber 向上找到 根节点。
 */
function scheduleWorkToRoot(fiber: Fiber, expirationTime): FiberRoot | null {
  // 做了一个记录，相当于什么都没做
  recordScheduleUpdate();

  // Update the source fiber's expiration time
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
  let root = null;
逼
  // 如果 node 等于 null 代表就是 RootFiber
  if (node === null && fiber.tag === HostRoot) {
    root = fiber.stateNode;
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

  // 跟踪应用更新
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

- 找到当前`Fiber`的 root
- 给更新节点的父节点链上的每个节点的`expirationTime`设置为这个`update`的`expirationTime`，除非他本身时间要小于`expirationTime`
- 给更新节点的父节点链上的每个节点的`childExpirationTime`设置为这个`update`的`expirationTime`，除非他本身时间要小于`expirationTime`

最终返回 root 节点的`Fiber`对象

找到根节点，同时更新所有节点的 childExpirationTime

```javascript
/*** react-reconciler/src/ReactFiberScheduler.js ***/

function resetStack() {
  // nextUnitOfWork 用于记录render阶段Fiber树遍历过程中下一个需要执行的节点。
  if (nextUnitOfWork !== null) { // 代表之前更新的是异步任务，并且执行到一半由于时间片不够，吧执行权给浏览器
    let interruptedWork = nextUnitOfWork.return;
    
    // 这样理论上也是找到根节点
    while (interruptedWork !== null) {
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

#### requestWork

- 加入到 root 调度队列
- 判断是否批量更新
- 根据 expirationTime 判断调度类型

```javascript
/*** react-reconciler/src/ReactFiberScheduler.js ***/

// requestWork is called by the scheduler whenever a root receives an update.
// It's up to the renderer to call renderRoot at some point in the future.
/**
 * @description 每当 root 收到更新时，调度程序都会调用 requestWork。取决于渲染器在将来的某个时刻调用renderRoot。
 * @param { scheduleWorkToRoot 找到的 FIberRoot } root
 * @param { 计算出来的过期时间 } expirationTime
 */
function requestWork(root: FiberRoot, expirationTime: ExpirationTime) {
  // 把当前 root 添加到调度里面，因为每次调度总是从 FiberRoot 开始执行
  addRootToSchedule(root, expirationTime);
  // 只有在 performWorkOnRoot 的时候和 commit 的时候是 true，其他时候都是 false
  if (isRendering) {
    // Prevent reentrancy. Remaining work will be scheduled at the end of
    // the currently rendering batch.
    // 防止重入。 剩余的工作将安排在当前渲染批次的末尾。
    return;
  }

  // 这里不是太懂批量更新，涉及到事件系统
  if (isBatchingUpdates) {
    // Flush work at the end of the batch.
    if (isUnbatchingUpdates) {
      // ...unless we're inside unbatchedUpdates, in which case we should
      // flush it now.
      // 除非我们在 unbatchedUpdates 内部，在这种情况下我们应该现在 flush 它。
      nextFlushedRoot = root;
      nextFlushedExpirationTime = Sync;
      performWorkOnRoot(root, Sync, false);
    }
    return;
  }

  // TODO: Get rid of Sync and use current time?
  // 如果是同步任务执行 performSysnWork
  if (expirationTime === Sync) {
    performSyncWork();
  } else {
    scheduleCallbackWithExpirationTime(root, expirationTime);
  }
}
```

```javascript
/*** react-reconciler/src/ReactFiberScheduler.js ***/

/**
 * @param { scheduleWorkToRoot 找到的 FIberRoot } root
 * @param { 计算出来的过期时间 } expirationTime
 */
function addRootToSchedule(root: FiberRoot, expirationTime: ExpirationTime) {
  // Add the root to the schedule.
  // Check if this root is already part of the schedule.
  // 表示现在调度链里面没有下一个 root，就只有一个 root 节点，那么就添加进去
  if (root.nextScheduledRoot === null) {
    // This root is not already scheduled. Add it.
    root.expirationTime = expirationTime;
    // 循环链接增加节点的过程
    if (lastScheduledRoot === null) {
      firstScheduledRoot = lastScheduledRoot = root;
      root.nextScheduledRoot = root;
    } else {
      lastScheduledRoot.nextScheduledRoot = root;
      lastScheduledRoot = root;
      lastScheduledRoot.nextScheduledRoot = firstScheduledRoot;
    }
  } else {
    // This root is already scheduled, but its priority may have increased.
    // 此 root 已经被调度，但其优先级可能已增加。
    const remainingExpirationTime = root.expirationTime;
    if (expirationTime > remainingExpirationTime) {
      // Update the priority.
      root.expirationTime = expirationTime;
    }
  }
}
```

