# React 思想学习

> 简单了解 React 思想再进行 React Native 的学习，React 和 React Native 在实际操作中还是有一定的差别。原文地址 [阮一峰-React 入门实例教程
> ](http://www.ruanyifeng.com/blog/2015/秘钥03/react.html)

* __ReactDOM.render()__

    * ReactDOM.render 是 React 的最基本方法，用于将模板转为 HTML 语言，并插入指定的 DOM 节点

* __JSX__

    * HTML 语言直接写在 JavaScript 语言之中，允许 HTML 与 JavaScript 混写
    * 基本语法规则：遇到 HTML 标签（以 `<` 开头），就用 HTML 规则解析；遇到代码块（以 `{` 开头），就用 JavaScript 规则解析

* __组件__

    * 允许将代码封装成组件（component），然后在网页中像插入 HTML 标签一样插入组件
    * 组件名称必须以大写字母开头
    * 必须拥有自己的 Render 方法用于输出组件
    * 只能含有一个顶层标签

* __this.props.children__

    * `this.props`  对象的属性与组件的属性一一对应，多了  `this.props.children` 属性用来表示组件的所有子节点
    * `this.props.children` 没有子节点时结果是 `undefined`，只有一个子节点时结果是 `object`，多个子节点时结果是 `array`
    * 使用 `React.Children.map` 来遍历子节点

* __PropTypes__

    * `PropTypes`  属性用来验证组件实例的属性是否符合要求
    * `getDefaultProps` 用来设置组件属性的默认值

* __获取真实的DOM节点__

    * 组件并不是真实的 DOM 节点而是存在内存中的一种数据结构，叫虚拟 DOM
    * 虚拟 DOM 只有插入文档以后才会变成真实的 DOM
    * 所有的 DOM 变化都是先发生在虚拟 DOM 上再将实际发生变动的部分反映在真实 DOM 上，这算法叫
       DOM diff，极大提升网页性能表现
    * 通过 `this.ref.[refName]` 获取真实 DOM 的节点，需要注意的是必须等虚拟 DOM 插入之后才能获取到这个属性

* __this.state__

    * 将组件看做状态机，状态变化触发重新渲染 UI
    * 与 `this.props` 的区别在于，`this.props` 表示定义后就不再改变的特性，而 `this.state` 表示会随着用户互动而产生变化的特性

* __表单__

    * 文本框的值不能用 `this.props.value` 读取
    * 定义一个 `onChange` 事件的回调函数，通过 `event.target.value` 读取用户输入的值
    * `textarea` 元素、`select元素`、`radio` 元素等都属于这种情况

* __组件的生命周期__

    * 生命周期为 3 个状态：

        * Mounting：已插入真实 DOM
        * Updating：正在被重新渲染
        * Unmounting：已移出真实 DOM

    * `will` 函数在进入状态之前调用，`did` 函数在进入状态之后调用，3 种状态共计 5 种处理函数：

        * `componentWillMount()`
        * `componentDidMount()`
        * `componentWillUpdate(object nextProps, object nextState)`
        * `componentDidUpdate(object prevProps, object prevState)`
        * `componentWillUnmount()`

    * 2 个特殊状态处理函数：

        * `componentWillReceiveProps(object nextProps)`：已加载组件收到新的参数时调用
        * `shouldComponentUpdate(object nextProps, object nextState)`：组件判断是否重新渲染时调用

* __Ajax__ 

    * 可在 `componentDidMount()` 中进行 Ajax 请求，成功后使用 `this.setState` 方法重新渲染 UI

