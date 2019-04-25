**核心知识点**

- 维护时间片
- 模拟 requestldleCallback
- 调度列表和超时判断

可参考文章：

[scheduler 源码详解](https://juejin.im/post/5c32c0c86fb9a049b7808665#comment)

[浅谈React Scheduler任务管理](https://zhuanlan.zhihu.com/p/48254036)

**scheduler调度算法**

首先，要明确几点：

- scheduler是用来做任务调度的

- 所有任务在一个调度生命周期内都有一个过期时间与调度优先级，但是调度优先级最终还是会转换为过期时间，只是过期时间长短的问题，过期时间越短代表越饥饿，优先级也就越高，但已经过期了的任务也会被视为饥饿任务

- requestAnimationFrameWithTimeout，这是React scheduler的一个超强的函数，它是解决网页选项卡如果在未激活状态下requestAnimationFrame不会被触发的问题，这样的话，调度器是可以在后台继续做调度的，一方面也能提升用户体验，同时后台执行的时间间隔是以100ms为步长，这个是一个最佳实践，100ms是不会影响用户体验同时也不影响CPU能耗的一个折中时间间隔

- 调度优先级分为：

- - 立即执行优先级，立即过期
  - 用户阻塞型优先级，250毫秒后过期
  - 空闲优先级，永不过期，可以在任意空闲时间内执行
  - 普通优先级，5秒后过期



**一个调度生命周期分为几个阶段**

- - 调度前

- - - 注册任务队列(环状链表，头接尾，尾接头)，按照过期时间从小到大排列，如果当前任务是最饥饿的任务，则排到最前面，并立即开始调度，如果并不是最饥饿的任务，则放到队列中间或者最后面，不做任何操作，等待被调度

  - 调度准备

  - - 通过requestAnimationFrame在下一次屏幕刚开始刷新的帧起点时计算当前帧的截止时间(33毫秒内)
    - 如果不超过当前帧的截止时间且当前任务没有过期，进入任务调度
    - 如果已经超过当前帧的截止时间，但没有过期，进入下一帧，并更新计算帧截止时间，重新判断时间(轮询判断)，直到没有任何过期超时或者超时才进入任务调度
    - 如果已经超过当前帧的截止时间，同时已经过期，进入过期调度

  - 正式调度

  - - 执行调度

    - - 在当前帧的截止时间前批量调用所有任务，不管是否过期

    - 过期调度

    - - 批量调用饥饿任务或超时任务的回调，删除任务节点

  - 调度完成

  - 检查任务队列是否还有任务

  - - 先执行最饥饿的任务
    - 如果存在任务，则进入下一帧，进入下一个调度生命周期



调度生命周期中，其实是可以被打断的，如果此时有一个优先级更高的任务进来，那么会重新执行**ensureHostCallbackIsScheduled** 方法，里面有一个**cancelHostCallback**方法，用来取消先前的调度，然后开始一个新的调度，当然原先被取消的调度也是会执行的，只不过是放在当前更高优先级任务的后面，这点在firstCallbackNode的结构中可以体现，它是一个通过previous和next首尾连接的闭环对象。



1、任务根据优先级和加入时的当前时间来确定过期时间

2、任务根据过期时间加入任务链表

3、任务链表有两种情况会启动任务的调度，1是任务链表从无到有时，2是任务链表加入了新的最高优先级任务时。

4、任务调度指的是在合适的时机去执行任务，这里通过`requestAnimationFrame`和`messageChannel`来模拟

5、`requestAnimationFrame`回调在帧首执行，用来计算当前帧的截止时间并开启递归，`messageChannel`的回调在帧末执行，根据当前帧的截止时间、当前时间、任务链表第一个任务的过期时间来决定当前帧是否执行任务（或是到下一帧执行）

6、如果执行任务，则根据任务是否过期来确定如何执行任务。任务过期的话就会把任务链表内过期的任务都执行一遍直到没有过期任务或者没有任务；任务没过期的话，则会在当前帧过期之前尽可能多的执行任务。最后如果还有任务，则回到第5步，放到下一帧再重新走流程。



```javascript
function scheduleCallbackWithExpirationTime(
  root: FiberRoot,
  expirationTime: ExpirationTime,
) {
  // 代表已经有任务在执行
  if (callbackExpirationTime !== NoWork) {
    // A callback is already scheduled. Check its expiration time (timeout).
    if (expirationTime < callbackExpirationTime) {
      // 当前的优先级低，直接 return
      // Existing callback has sufficient timeout. Exit.
      return;
    } else {
      if (callbackID !== null) {
        // Existing callback has insufficient timeout. Cancel and schedule a
        // new one.
        cancelDeferredCallback(callbackID);
      }
    }
    // The request callback timer is already running. Don't start a new one.
  } else {
    // prolfilly相关的
    startRequestCallbackTimer();
  }

  callbackExpirationTime = expirationTime;
  // originalStartTimeMs 是代码加载的时候获取的时间
  const currentMs = now() - originalStartTimeMs;
  // 下面这段代码没看懂想要干啥
  const expirationTimeMs = expirationTimeToMs(expirationTime);
  const timeout = expirationTimeMs - currentMs;
  callbackID = scheduleDeferredCallback(performAsyncWork, {timeout});
}
```



```javascript
// 第一个是要执行的callback,暂时可以理解为一个任务。第二个参数是可选的，可以传入一个超时时间来标识这个任务过多久超时。如果不传的话就会根据上述的任务优先级确定过期时间。
// 在 React 里 callback 其实就是 performAsyncWork
function unstable_scheduleCallback(callback, deprecated_options) {
  var startTime =
    currentEventStartTime !== -1 ? currentEventStartTime : getCurrentTime();

  var expirationTime;
  if (
    typeof deprecated_options === 'object' &&
    deprecated_options !== null &&
    typeof deprecated_options.timeout === 'number'
  ) {
    // FIXME: Remove this branch once we lift expiration times out of React.
    expirationTime = startTime + deprecated_options.timeout;
  } else {
    switch (currentPriorityLevel) {
      case ImmediatePriority:
        expirationTime = startTime + IMMEDIATE_PRIORITY_TIMEOUT;
        break;
      case UserBlockingPriority:
        expirationTime = startTime + USER_BLOCKING_PRIORITY;
        break;
      case IdlePriority:
        expirationTime = startTime + IDLE_PRIORITY;
        break;
      case LowPriority:
        expirationTime = startTime + LOW_PRIORITY_TIMEOUT;
        break;
      case NormalPriority:
      default:
        expirationTime = startTime + NORMAL_PRIORITY_TIMEOUT;
    }
  }

  var newNode = {
    callback,
    priorityLevel: currentPriorityLevel,
    expirationTime,
    next: null,
    previous: null,
  };


  // Insert the new callback into the list, ordered first by expiration, then
  // by insertion. So the new callback is inserted any other callback with
  // equal expiration.
  // 上面的代码根据入参或者当前的优先级来确定当前callback的过期时间，并生成一个真正的任务节点。接下来就要把这个节点按照expirationTime排序插入到任务的链表里边去。
    
  // 下面是按照 expirationTime 把 newNode 加入到任务队列里
  if (firstCallbackNode === null) {
    // This is the first callback in the list.
    firstCallbackNode = newNode.next = newNode.previous = newNode;
    ensureHostCallbackIsScheduled();
  } else {
    var next = null;
    var node = firstCallbackNode;
    do {
      if (node.expirationTime > expirationTime) {
        // The new callback expires before this one.
        next = node;
        break;
      }
      node = node.next;
    } while (node !== firstCallbackNode);

    if (next === null) {
      // No callback with a later expiration was found, which means the new
      // callback has the latest expiration in the list.
      next = firstCallbackNode;
    } else if (next === firstCallbackNode) {
      // The new callback has the earliest expiration in the entire list.
      firstCallbackNode = newNode;
      ensureHostCallbackIsScheduled();
    }

    var previous = next.previous;
    previous.next = next.previous = newNode;
    newNode.next = next;
    newNode.previous = previous;
  }

  return newNode;
}
```



// reactscheduler 3

```javascript
function ensureHostCallbackIsScheduled() {
  // 代表 performSyncWork 已经在执行了
  if (isExecutingCallback) {
    // Don't schedule work yet; wait until the next time we yield.
    return;
  }
  // Schedule the host callback using the earliest expiration in the list.
  var expirationTime = firstCallbackNode.expirationTime;
  if (!isHostCallbackScheduled) {
    isHostCallbackScheduled = true;
  } else {
    // Cancel the existing host callback.
    cancelHostCallback();
  }
  requestHostCallback(flushWork, expirationTime);
}
```

```javascript
  requestHostCallback = function(callback, absoluteTimeout) {
    scheduledHostCallback = callback;
    timeoutTime = absoluteTimeout;
    if (isFlushingHostCallback || absoluteTimeout < 0) {
      // Don't wait for the next frame. Continue working ASAP, in a new event.
      port.postMessage(undefined);
    } else if (!isAnimationFrameScheduled) {
      // If rAF didn't already schedule one, we need to schedule a frame.
      // TODO: If this rAF doesn't materialize because the browser throttles, we
      // might want to still have setTimeout trigger rIC as a backup to ensure
      // that we keep performing work.
      isAnimationFrameScheduled = true;
      requestAnimationFrameWithTimeout(animationTick);
    }
  };
```

