# React学习之初入React世界

* ### JSX语法
> 1. JSX将HTML语法直接加入到JavaScript代码中，再通过翻译器装换到纯JavaScript后由浏览器执行。在实际开发中，JSX在产品打包阶段都已经编译成纯JavaScript，不会带来任何副作用，反而会让代码更加直观并易于维护。
> 2. 事实上，JSX并不需要花精力学习。只要熟悉HTML标签，大多数功能就都可以直接使用了。
> 3. JSX语法完美地利用了JavaScript自带的语法和特性，并使用大家熟悉的HTML语法来创建虚拟元素。可以说，JSX基本语法基本被XML囊括了，但也有少许不同。
* ### React组件
> 1. 基本的封装性。
> 2. 简单的生命周期呈现。
> 3. 明确的数据流动。
* ### React组件的构建方法
> 1. React.createClass
> 2. ES6 classes
> 3. 无状态函数
* ### React数据流
>在React中，数据是自顶向下单向流动的，即从父组件到子组件。这条原则让组件之间的关系变得简单且可预测。state与props是React组件中最重要的概念。如果顶层组件初始化props，那么React会向下遍历整颗组件树，重新尝试渲染所有相关的子组件。而state只关心每个组件自己内部的状态，这些状态只能在组件内改变。把组件看成一个函数，那么它接受了props作为参数，内部由state作为函数的内部参数，返回一个Virtual DOM的实现。
* ### React生命周期
> 1. 组件的挂载与卸载：componentWillMount方法会在render方法之前执行，而componentDidMount方法会在render方法之后执行，分别代表了渲染前后的时刻。还有componentWillUnmount代表组件卸载前的状态，在这个方法中，我们常常会执行一些清理方法，如事件回收或是清除定时器。
> 2. 更新过程是指父组件向下传递props或组件自身执行setState方法时发生的一系列更新动作。shouldComponentUpdate是一个特别的方法，它接受需要更新的props和state，让开发者增加必要的条件判断，让其在需要时更新，不需要时不更新。因此，当方法返回false的时候，组件不再向下执行生命周期方法。componentWillUpdate和componentDidUpdate这两个生命周期方法很容易理解，对应的初始化方法也很容易知道，他们代表在更新过程中渲染前后的时刻。
> * ## getDerivedStateFromProps(props, state)
> 在 render() 之前触发，不管是什么原因触发 render() 方法的,此方法应返回一个对象，用于更新 State, 或返回 null 不更新。
> * ## getSnapshotBeforeUpdate(prevProps, prevState)
> 在 Dom 改变之前获得一些最新的信息,此方法的一切返回值都将被传递给 componentDidUpdate 方法中的 snapshot 参数。
> * ## componentDidCatch(err, info)
> 1. catch js 错误，log 这些 errors，显示一个回调的 UI。
> 2. 获取错误的时机：during rendering，生命周期函数和子组件的 constructor 函数。
> 3. 使用 setState() 获取 unhandled JS errors 和 显示回调 UI。
* ### ReactDOM
> 1. findDOMNode:React提供的获取DOM元素的方法有两种，其中一种就是ReactDOM提供的findDOMNode，当组件被渲染到DOM中，findDOMNode返回该React组件实例相应的DOM节点。
> 2. 为什么说只有在顶层组件我们才不得不使用ReactDOM呢？这是因为要把React渲染的Virtual DOM渲染到浏览器的DOM当中，就要使用render方法了，该方法把元素挂载到container中，并且返回element的实例。
* ### refs

> 在组件内，JSX是不会返回一个组件的实例的，它只是一个ReactElement，只是告诉React被挂载的组件应该长什么样。refs就是为此而生的，它是React组件中非常特殊的prop，可以附加到任何一个组件上。从字面意思来看，refs即reference，组件被调用时会新建一个该组件的实例，而refs就会指向这个实例，它可以是一个回调函数，这个回调函数会在组件被挂载后立即执行。