在上一篇将 React Fiber 架构中，已经说到过，React 现在将整体的数据结构从树改为了链表结构。

那么就需要你掌握链表相关的基本操作，如果不熟悉链表的操作，请先熟悉链表的操作。



而链表的每一个节点是 Fiber，而不是在 16 之前的虚拟DOM 节点。

> 我这里说的虚拟 DOM 节点是指 React.createElement 方法所产生的节点。虚拟 DOM tree 只维护了组件状态以及组件与 DOM 树的关系，Fiber Node 承载的东西比 虚拟 DOM 节点多很多。



**了解 Diff**

我们知道 Diff 就是新旧节点的对比，在上文中也说道了，这里面的 Diff 主要是构建 currentInWorkProgress 的过程，同时得到 Effect List，给下一个阶段 commit 做准备。

是一层一层的比较。

## Diff

对于 Diff， 就是新老节点的对比，我们以新节点为标准，然后来构建整个 currentInWorkProgress，对于新的 children 会有四种情况。

- TextNode(包含字符串和数字)
- 单个 React Element(通过该节点是否有 $$typeof 区分)
- 数组
- 可迭代的 children，跟数组的处理方式差不多

那么我们就来一步一步的看这四种类型是如何进行 diff 的。

### Diff TextNode

首先看 TextNode，因为它是最简单的，担心直接看到难的，然后就打击你的信心。

```javascript
// demo1：当前 ui 对应的节点的 jsx
return (
  <div>
  // ...
  	<div>
  		<xxx></xxx>
  		<xxx></xxx>
  	</div>
  //...
	</div>
)

// demo2：更新成功后的节点对应的 jsx

return (
  <div>
  // ...
  	<div>
  		前端桃园
  	</div>
  //...
	</div>
)

```

对应的单链表结构图：

![image-20190714223931338](http://imgs.taoweng.site/2019-07-14-143931.png)



对于 diff TextNode 会有两种情况。

1. currentFirstNode 是 TextNode
2. currentFirstNode 不是 TextNode

> currentFirstNode

**为什么要分两种情况呢？**原因就是为了复用节点

**第一种情况**。xxx 是一个 TextNode，那么就代表这这个节点可以复用，有复用的节点，对性能优化很有帮助。既然新的 child 只有一个 TextNode，那么复用节点之后，就把剩下的 aaa 节点就可以删掉了，那么 div 的 child 就可以添加到 workInProgress 中去了。

源码如下：

```javascript
if (currentFirstChild !== null && currentFirstChild.tag === HostText) {
      // We already have an existing node so let's just update it and delete
      // the rest.
      deleteRemainingChildren(returnFiber, currentFirstChild.sibling);
      const existing = useFiber(currentFirstChild, textContent, expirationTime);
      existing.return = returnFiber;
      return existing;
}
```

在源码里 `useFiber` 就是复用节点的方法，`deleteRemainingChildren` 就是删除剩余节点的方法，这里是从 `currentFirstChild.sibling` 开始删除的。

**第二种情况。**xxx 不是一个 TextNode，那么就代表这个节点不能复用，所以就从 `currentFirstChild`开始删掉剩余的节点，对应到上面的图中就是删除掉 xxx 节点和 aaa 节点。

对于源码如下：

```javascript
deleteRemainingChildren(returnFiber, currentFirstChild);
const created = createFiberFromText(
    textContent,
    returnFiber.mode,
    expirationTime,
);
created.return = returnFiber;
```

其中 `createFiberFromText` 就是根据 `textContent` 来创建节点的方法。

> 注意：删除节点不会真的从链表里面把节点删除，只是打一个 delete 的 tag，当 commit 的时候才会真正的去删除。

### Diff React Element

有了上面 TextNode 的 Diff 经验，那么来理解 React Element 的 Diff 就比较简单了，因为他们的思路是一致的：先找有没有可以复用的节点，如果没有就另外创建一个。

那么就有一个问题，**如何判断这个节点是否可以复用呢？**

有两个点：1. key 相同。 2. 节点的类型相同。

如果以上两点相同，就代表这个节点只是变化了内容，不需要创建新的接口，可以复用的。

对应的源码如下：

```javascript
if (child.key === key) {
  if (
    child.tag === Fragment
    ? element.type === REACT_FRAGMENT_TYPE
    : child.elementType === element.type
  ) {
    // 为什么要删除老的节点的兄弟节点？
    // 因为当前节点是只有一个节点，而老的如果是有兄弟节点是要删除的，是多于的。删掉了之后就可以复用老的节点了
    deleteRemainingChildren(returnFiber, child.sibling);
    // 复用当前节点
    const existing = useFiber(
      child,
      element.type === REACT_FRAGMENT_TYPE
      ? element.props.children
      : element.props,
      expirationTime,
    );
    existing.ref = coerceRef(returnFiber, child, element);
    existing.return = returnFiber;
    return existing;
}
```

相信这些代码都很好理解了，除了判断条件跟前面 TextNode 的判断条件不一样，其余的基本都一样，只是 React Element 多了一个跟新 ref 的过程。

同样，如果节点的类型不相同，就将节点从当前节点开始把剩余的都删除。

```javascript
deleteRemainingChildren(returnFiber, child);
```

到这里，可能你们就会觉得接下来应该就是讲解当没有可以复用的节点的时候是如果创建节点的。

不过可惜你们猜错了。因为 Facebook 的工程师很厉害，另外还做了一个工作来优化，来找到复用的节点。

我们现在来看这种情况：

![image-20190714232052778](/Users/licai/Library/Application Support/typora-user-images/image-20190714232052778.png)

这种情况就是有可能更新的时候删除了一个节点，但是另外的节点还留着。

那么在对比 xxx 节点和 AAA 节点的时候，它们的节点类型是不一样，按照我们上面的逻辑，还是应该把 xxx 和 AAA 节点删除，然后创建一个 AAA 节点。

但是你看，明明 xxx 的 slibling 有一个 AAA 节点可以复用，但是被删了，多浪费呀。所以还有另外有一个策略来找 xxx 的所有兄弟节点中有没有可以复用的节点。

这种策略就是从 div 下面的所有子节点去找有没有可以复用的节点，而不是像 TextNode 一样，只是找第一个 child 是否可以复用，如果当前节点的 key 不同，就代表肯定不是同一个节点，所以把当前节点删除，然后再去找当前节点的兄弟节点，直到找到 key 相同，并且节点的类型相同，否则就删除所有的子节点。

> 你有木有这样的问题：为什么 TextNode 不采用这样的循环策略来找可以复用的节点呢？这个问题留给你思考，欢迎在评论区留下你的答案。

对应的源码逻辑如下：

```javascript
// 找到 key 相同的节点，就会复用当前节点
while (child !== null) {
  if (child.key === key) {
    if (
      child.tag === Fragment
      ? element.type === REACT_FRAGMENT_TYPE
      : child.elementType === element.type
    ) {
      // 复用节点逻辑，省略该部分代码，和上面复用节点的代码相同
      // code ...
      return existing;
    } else {
      deleteRemainingChildren(returnFiber, child);
      break;
    }
  } else {
    // 如果没有可以复用的节点，就把这个节点删除
    deleteChild(returnFiber, child);
  }
  child = child.sibling;
}
```

在上面这段代码我们需要注意的是，当 key 相同，React 会认为是同一个节点，所以当 key 相同，节点类型不同的时候，React 会认为你已经把这个节点重新覆盖了，所以就不会再去找剩余的节点是否可以复用。只有在 key 不同的时候，才会去找兄弟节点是否可以复用。

接下来才是我们前面说的，如果没有找到可以复用的节点，然后就重新创建节点，源码如下：

```javascript
// 前面的循环已经把该删除的已经删除了，接下来就开始创建新的节点了
if (element.type === REACT_FRAGMENT_TYPE) {
  const created = createFiberFromFragment(
    element.props.children,
    returnFiber.mode,
    expirationTime,
    element.key,
  );
  created.return = returnFiber;
  return created;
} else {
  const created = createFiberFromElement(
    element,
    returnFiber.mode,
    expirationTime,
  );
  created.ref = coerceRef(returnFiber, currentFirstChild, element);
  created.return = returnFiber;
  return created;
}
```

对于 Fragment 节点和一般的 Element 节点创建的方式不同，因为 Fragment 本来就是一个无意义的节点，他真正需要创建 Fiber 的是它的 children，而不是它自己，所以 `createFiberFromFragment` 传递的不是 element ，而是 element.props.children。

### Diff Array





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



