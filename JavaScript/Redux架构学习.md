# Redux 架构学习

> 据阮一峰说，React 并不是 Web 应用的完整解决方案，它没有涉及 __代码结构__ 和 __组件之间通信__ ，无法驾驭大型应用，所以使用 __Redux 架构__ 来进行补充。原文所在地址： 
> * [Redux 入门教程（一）：基本用法](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_one_basic_usages.html)
> * [Redux 入门教程（二）：中间件与异步操作](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_two_async_operations.html)
> * [Redux 入门教程（三）：React-Redux 的用法](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_three_react-redux.html)

* __Redux 的适用场景__

    * 从结构角度上：

        * 用户的使用方式复杂 
        * 不同身份的用户有不同的使用方式（比如普通用户和管理员)
        * 多个用户之间可以协作
        * 与服务器大量交互，或者使用了WebSocket
        * View要从多个来源获取数据

    * 从组件角度上：

        * 某个组件的状态，需要共享 
        * 某个状态需要在任何地方都可以拿到
        * 一个组件需要改变全局状态
        * 一个组件需要改变另一个组件的状态

* __设计思想__

    * Web 应用是一个状态机，视图与状态是一一对应的
    * 所有的状态，保存在一个对象里面

* __基本概念和 API__

    * Store

        * 用来保存数据，整个应用只有一个
        * `createStore` 用来生成 Store，可传入 3 个参数，第 1 个参数是 Reducer 函数，第 2 个参数是 State 的初始状态，第 3 个参数是使用 applyMiddleware 传入中间件

    * State 

        * 对 Store 生成快照得到某个时点的数据
        * `store.getState()` 拿到当前时点的数据

    * Action

        * State 变化会导致 View 的变化，用户只能接触到 View，Action 即是 View 发出的表示 State 发生变化的通知
        * Action 对象的 `type` 属性是必须的，标识名称，其他属性任意设置
        * Action 是改变 State 的唯一办法，它会运送数据到 Store
        * 自定义的生成 Action 的函数即是 Action Creator
        * `store.dispatch()` 接受一个 Action 对象作为参数，是 View 发出 Action 的唯一方法

    * Reducer

        * Store 收到 Action 后 给出新的 State 使得 View 发生变化, 这个计算 State 的过程即是 Reducer
        * Reducer 函数接受 Action 和当前 State 作为参数，返回新 State
        * `store.dispatch` 会触发 Reducer 的自动执行，当然，需要将自己的 Reducer 函数在 `createStore()` 这一步骤作为参数传入
        * Reducer 是纯函数，所以 Reducer 函数中不得去改变 State，而是每次生成一个新对象

            * 不得改写参数
            * 不能调用系统 I/O 的API
            * 不能调用 `Date.now()` 或者 `Math.random()` 等不纯的方法，因为每次会得到不一样的结果

        * `store.subscribe()` 可监听 state 变化状态，可把 View 的更新函数（即组件的 render 或 `setState()`），就可实现自动渲染
        * 使用 `combineReducer` 方法可将多个 Reducer 函数合成一个，这用于分拆很有效

    * 流程梳理：

        1. 用户发出 Action ，`store.dispath(action)`  
        2. Store 自动调用 Reducer，传入当前 State 和 Action，Reducer 返回新的 State
        3. State 发生变化，Store 调用监听函数，`store.subscribe(listener)`，listener 通过 `store.getState()` 得到当前状态，可以选择重新渲染 View

* __中间件和异步操作__

    * 在 `createStore` 时可使用 `applyMiddleware` 传入中间件，思想类似于面向切面编程，是在发送 Action 这一步骤的前后插入方法实现其他通用操作，例如打印日志
    * 异步操作的实现：

        * 可以发送一个返回函数的 Action Creator，但是 `store.dispatch ` 的参数只支持对象不支持函数，可用 `redux-thunk` 中间件改造出支持函数的 `store.dispatch`
        * 使用 `redux-promise` 实现支持接受 `promise` 的 `store.dispatch`
        * 使用 `redux-actions` 的 `createAction` 方法 创建一个 `promise` 作为 Action 的 `payload` 属性值

* __React-Redux__

    * UI 组件

        * 只负责 UI 的呈现，不带有任何业务逻辑
        * 没有状态（即不使用 `this.state` 这个变量）
        * 所有数据都由参数（`this.props`）提供
        * 不使用任何 Redux 的 API

    * 容器组件 

        * 负责管理数据和业务逻辑，不负责 UI 的呈现
        * 带有内部状态
        * 使用 Redux 的 API

    * connect()

        * 用于从 UI 组件生成容器组件
        * 参数 `mapStateToProps`，输入逻辑：外部的数据（即state对象）如何转换为 UI 组件的参数 

            * 建立外部的 `state` 对象到 UI 组件的 `props` 对象的映射关系
            * 是一个函数，接受 `state` 对象作为参数，返回一个对象
            * 会订阅 Store，当 `state` 更新时会自动执行，重新计算 UI 组件的参数触发重新渲染
            * 也可以使用容器组件的 `props` 对象，但这也会引起重新渲染
            * `connect` 方法可以省略 `mapStateToProps` 参数，但这样 UI 组件就不会订阅 Store，所以就不会被 Store 的更新引起重绘了

        * 参数 `mapDispatchToProps`，输出逻辑：用户发出的动作如何变为 Action 对象，从 UI 组件传出去

            * 建立 UI 组件的参数到 `store.dispatch` 的映射，定义了用户的哪些操作应当当做 Action 传给 Store，可以是函数也可以是对象
            * 若是函数，会得到 `dispatch` 和 `ownProps`（容器组件的 `props` 对象）两个参数，返回对象，其中定义一系列的映射来表达 UI 组件的参数如何发 Action
            * 若是对象，每个键名也是对应 UI  组件的同名参数，键值应该是一个函数，会被当作 Action creator ，返回的 Action 会由 Redux 自动发出

    * Providor 组件

        * 作用是让容易组件拿到 `state`
        * 使用 `Provider` 在根组件外面包一层，`APP` 所有子组件就默认能获得 `state` 了，这种方式是将 store 放在了 `React` 组件的 `context` 属性上 
