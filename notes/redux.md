>  所有依赖 props ，都应该放到 componentWillReceiveProps 这个生命周期里。

为什么这句话我要写到 redux 里，因为可能在有些事情是在 dispatch 之后再去获取 store 里面的数据，然后再那到这个数据再去做一些事情，那么这个事情，就应该放到 componentWillReceiveProps 里。

>  在涉及到 redux 的 react 应用，进行事件处理的时候，一般 handle 事件只需要对数据进行 format ，然后 dispatch 就行了，尽量少 setState。

因为我们在使用 redux 的时候，只要 connect 了的，那么我们这个组件的状态应该是 redux 来管理，那么状态变更就需要通过 redux 来管理了，而不是 react，所以我们应该在 容器组件里面少用 state。

注意少用而不是不用，毕竟对于像控制 模态框显示的这种状态，我感觉还是通过 state 管理比较好。没必要放到 redux 里进行管理。

所以在一个 容器组件里，基本上都是 各种 dispatch 而不是各种 setState ,这才是一个合格的redux 应用应该有的状态。