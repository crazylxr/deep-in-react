

## Fiber

- React 16 的调和引擎
- 关键特性：时间分片(time slicing)和暂停(supense)

也是一种数据结构



###  一个 Fiber 是一个任务分片

- 暂停当前工作任务，并稍后返回继续
- 分片不同的优先级给不同的任务
- 复用任务
- 丢弃/终止任务

### 如何实现

遍历方式就是 workloop 函数

###  Fiber 工作流程

current vs workInProgress

- current tree：反映当前屏幕的 UI。HostRoot.current
- workInProgress tree：反映将要变化的 UI。current.alternate
- 两棵树上对应的Fiber Node 通过alternate 属性，互相引用

![image-20190404164810369](/Users/licai/Library/Application Support/typora-user-images/image-20190404164810369.png)

### Effect

- Effect 定义了 Fiber Node 在 commit 阶段要做的事情
- 对于组件，例如更新组件ref，调用componentDidMount、componentDidUpdate等
- 对于DOm阶段，例如DOM 的增加、更新、删除等
- 字段名：effectTag

组成 effects list



## React 组件更新细节

### React 基本工作流程

Schedule -> Reconciliation/Render -> Commit -> Browser Screen



#### Schedule阶段

setState -> enqueueUpdate ->scheduleWork -> requestWork

#### render 阶段

- beginWork ：根据Fiber Node 的 tag，将逻辑代理到对应的函数处理，如 tag 为 ClassComponent，由 updateClassComponent 处理
- updateXXX：如updateClassComponent ，借助组件实例，执行如下操作：
  1. UNSAFE_componentWillReceiveProps
  2. 执行 updateQueue 中的更新，计算新的 state
  3. getDerivedStateFromProps
  4. shouldComponentUpdate
  5. UNSAFE_componentWillUpdate
  6. 增加一个 effect (componentDidUpdate)
  7. 更新组件实例的 state 和 props
  8. render(reconcile children)

#### Commit 阶段

- commitRoot：使effects list 生效。同步执行，不能被打断
  1. commitBeforeMutaionLifeCycles：getSnapshotBeforeUpdate
  2. commitAllHostEffects：更新 DOM
  3. commitAllLifeCycles：componentDidUpdate/componentDidMount

## 并行渲染

![image-20190404175217869](/Users/licai/Library/Application Support/typora-user-images/image-20190404175217869.png)

### 优先级

- 通过 expirationTime确定优先级，优先级体现在 3 个维度上
  - HostRoot
  - Fiber Node
  - update queue