# React学习之漫谈React

* ### 事件系统
> 1. 合成事件的绑定方式
>     `<button onClick={this.handleClick}>Test</button>`
> 2. 合成事件的实现机制：事件委派和自动绑定。
> 3. React合成事件系统的委托机制，在合成事件内部仅仅是对最外层的容器进行了绑定，并且依赖事件的冒泡机制完成了委派。
* ### 表单
> 1. React受控组件更新state的流程：
> > 1. 可以通过在初始state中设置表单的默认值。
> > 2. 每当表单的值发生变化时，调用onChange事件处理器。
> > 3. 事件处理器通过合成事件对象e拿到改变后的状态，并更新应用的state。
> > 4. setState触发视图的重新渲染，完成表单组件值的更新。
>
> 2. 受控组件和非受控组件的最大区别是：非受控组件的状态并不会受应用状态的控制，应用中也多了局部组件状态，而受控组件的值来自于组件的state。
* ### 样式处理
> 1. CSS模块化遇到了哪些问题？全局污染，命名混乱，依赖管理不彻底，无法共享变量，代码压缩不彻底。
> 2. CSS Modules模块化方案:启用CSS Modules，样式默认局部，使用composes来组合样式。
* ### 组件间通信
> 1. 子组件向父组件通信
> > 1. 利用回调函数
> > 2. 利用自定义事件机制
> 2. 当需要让子组件跨级访问信息时，我们还可以使用context来实现跨级父子组件间的通信。
> 3. 没有嵌套关系的组件通信：我们在处理事件的过程中需要注意，在componentDidMount事件中，如果组件挂载完成，再订阅事件；当组件卸载的时候，在componentWillUnmount事件中取消事件的订阅。
* ### 组件间抽象
> 1. mixin 的目的，就是为了创造一种类似多重继承的效果，或者说，组合。实际上，包括C++等一些年龄较大的OOP语言，都有一个强大但是危险的多重继承特性。现代语言权衡利弊，大都舍弃了它，只采用单继承。但是单继承在实现抽象的时候有很多不便，为了弥补缺失，Java引入接口（interface），其他一些语言则引入了mixin的技巧。
> > 封装mixin方法
> > 方法：
> > ```
> > const mixin = function(obj, mixins) {
> >  const newObj = obj;
> >  newObj.prototype = Object.create(obj.prototype);
> >  for (let prop in mixins) {
> >      if (mixins.hasOwnProperty(prop)) {
> >          newObj.prototype[prop] = mixins[prop]; 
> >      }
> >  }
> >  return newObj; 
> > }
> > ```
> > 应用：
> > ```
> > const BigMixin = { 
> >  fly: () => {
> >      console.log('I can fly'); 
> >  }
> > };
> > const Big = function() { 
> >  console.log('new big');
> > };
> > const FlyBig = mixin(Big, BigMixin);
> > const flyBig = new FlyBig(); // => 'new big'
> > flyBig.fly(); // => 'I can fly'
> > ```
> > 上面这段代码实现对象混入的方法是：用赋值的方式将mixin对象里的方法都挂载到原对象上。
> 2. 在React中使用mixin
> > React在使用createClass构建组件时提供了mixin属性，比如官方封装的：PureRenderMixin。
> > ```
> > import React from 'react';
> > import PureRenderMixin from 'react-addons-pure-render-mixin';
> > React.createClass({
> >  mixins: [PureRenderMixin],
> >  render() {
> >      return <div>foo</div>;
> >  }
> > });
> > ```
> > 在createClass对象参数中传入数组mixins，里面封装了我们需要的模块。mixins数组也可以添加多个mixin。同时，在React中不允许出现重名普通方法的mixin。而如果是生命周期方法，则React将会将各个模块的生命周期方法叠加在一起然后顺序执行。
> > 使用createClass实现的mixin为组件做了两件事： 
> > > 1. 工具方法：这是mixin的基本功能，如果希望共享一些工具类的方法，就可以直接定义它们然后在组件中使用。 
> > > 2. 生命周期继承，props和state合并。mixin能够合并生命周期方法。如果有很多mixin来定义componentDidMount这个周期，那么React会很机智的将它们都合并起来执行。同样，mixin也可以作state和props的合并。
> 3. ES6 Classes和decorator
> > 然而，当我们使用ES6 classes的形式构建组件的时候，却并不支持mixin。为了使用这个强大的功能，我们还需要采取其他方法，来达到模块重用的目的。可以使用ES7的语法糖decorator来实现class上的mixin。core-decorators库为开发者提供了一些实用的decorator, 其中也正好实现了我们想要的@mixin。
> > ```
> > import React, { Component } from 'React'; 
> > import { mixin } from 'core-decorators';
> > const PureRender = { 
> >  shouldComponentUpdate() {}
> > };
> > const Theme = { 
> >  setTheme() {}
> > };
> > @mixin(PureRender, Theme)
> > class MyComponent extends Component {
> >  render() {} 
> > }
> > ```
> > mixin的问题
> > > 1. 破坏了原有组件的封装：mixin会混入方法，给原有的组件带来新特性。但同时它也可能带来新的state和props，这意味着组件有一些“不可见”的状态需要我们去维护。另外，mixin也有可能去依赖其他的mixin，这样会建立一个mixin的依赖链，当我们改动一个mixin的状态，很有可能也会影响其他的mixin。
> > > 2. 命名冲突
> > > 3. 增加复杂性
> > > 针对这些困扰，React提出的新的方式来取代mixin，那就是高阶组件。
> 4. 高阶组件
> > 如果已经理解高阶函数，那么理解高阶组件也很容易的。高阶函数：就是一种这样的函数，它接受函数作为参数输入，或者将一个函数作为返回值。例如我们常见的方法map, reduce, sort等都是高阶函数。高阶组件和和高阶函数很类似，高阶组件就是接受一个React组件作为参数输入，输出一个新的React组件。高阶组件让我们的代码更具有复用性、逻辑性与抽象性，它可以对render方法作劫持，也可以控制props和state。
> > 实现高阶组件的方法有如下两种： 
> > > 1. 属性代理：高阶组件通过被包裹的React组件来操作props。 
> > > 2. 反向继承：高阶组件继承于被包裹的React组件。
> > > 属性代理
> > > 示例代码：
> > ```
> > import React, { Component } from 'React';
> > const MyContainer = (WrappedComponent) => 
> >  class extends Component {
> >      render() {
> >          return <WrappedComponent {...this.props} />;
> >      } 
> >  }
> > ```
> > 在代码中我们可以看到，render方法返回了传入的WrappedComponent组件。这样，我们就可以通过高阶组件来传递props。这种方式就是属性代理。
> > 如何使用上面这个高阶组件：
> > ```
> > import React, { Component } from 'React';
> > class MyComponent extends Component { 
> >  // ...
> > }
> > export default MyContainer(MyComponent);
> > ```
> > 这样组件就可以一层层的作为参数被调用，原始组件久具备了高阶组件对它的修饰。这样，保持单个组件封装的同时也保留了易用行。
> > 从功能上， 高阶组件一样可以做到像mixin对组件的控制：
> > > 1. 控制props
> > > 我们可以读取、增加、编辑或是移除从WrappedComponent传进来的props。
> > > 例如：新增props
> > > ```
> > > import React, { Component } from 'React';
> > > const MyContainer = (WrappedComponent) => 
> > >  class extends Component {
> > >      render() {
> > >          const newProps = {  text: newText, };
> > >          return <WrappedComponent {...this.props} {...newProps} />; 
> > >      }
> > >  }
> > > ```
> > > 注意：
> > > ```
> > > <WrappedComponent {...this.props}/>
> > > // is equivalent to
> > > React.createElement(WrappedComponent, this.props, null)
> > > ```
> > > 这样，当调用高阶组件的时候，就可以使用text这个新的props了。
> > > 2. 通过refs使用引用
> > > 3. 抽象state
> > > 高阶组件可以讲原组件抽象为展示型组件，分离内部状态。
> > > ```
> > > const MyContainer = (WrappedComponent) => 
> > >  class extends Component {
> > >      constructor(props) { 
> > >          super(props); 
> > >          this.state = { name: '', 4 };
> > >          this.onNameChange = this.onNameChange.bind(this); 
> > >      }
> > > 
> > >      onNameChange(event) { 
> > >          this.setState({
> > >              name: event.target.value, 
> > >          })
> > >      }
> > >      render() {
> > >          const newProps = {
> > >              name: {
> > >                  value: this.state.name, 
> > >                  onChange: this.onNameChange,
> > >              }, 
> > >          }
> > >          return <WrappedComponent {...this.props} {...newProps} />; 
> > >      }
> > >  }
> > > ```
> > > 在这个例子中，我们把组件中对name prop 的onChange方法提取到高阶组件中，这样就有效的抽象了同样的state操作。
> > > 使用方式
> > > ```
> > > @MyContainer
> > > class MyComponent extends Component {
> > >  render() {
> > >      return <input name="name" {...this.props.name} />;
> > >  } 
> > > }
> > > ```
> > > 反向继承
> > ```
> > const MyContainer = (WrappedComponent) => 
> > class extends WrappedComponent {
> >      render() {
> >          return super.render();
> >      } 
> >  }
> > ```
* ### 组件性能优化
> #### 性能优化的思路
> 影响网页性能最大的因素是浏览器的重排(repaint)和重绘(reflow)。React的Virtual DOM就是尽可能地减少浏览器的重排和重绘。从React渲染过程来看，如何防止不必要的渲染是解决问题的关键。
> #### 性能优化的具体办法
> > 1. 尽量多使用无状态函数构建组件
> > 无状态组件只有props和context两个参数。它不存在state，没有生命周期方法，组件本身即有状态组件构建方法中的render方法。在合适的情况下，都应该必须使用无状态组件。无状态组件不会像React.createClass和ES6 class会在调用时创建新实例，它创建时始终保持了一个实例，避免了不必要的检查和内存分配，做到了内部优化。
> > 2. 拆分组件为子组件，对组件做更细粒度的控制
> > ##### 相关重要概念:纯函数
> > 纯函数的三大构成原则:
> > > * 给定相同的输入，它总是返回相同的输出： 比如反例有 Math.random(), New Date();
> > > * 过程没有副作用：即不能改变外部状态;
> > > * 没有额外的状态依赖：即方法内部的状态都只能在方法的生命周期内存活，这意味着不能在方法内使用共享的变量。
> > > 纯函数非常方便进行方法级别的测试及重构，它可以让程序具有良好的扩展性及适应性。纯函数是函数式变成的基础。React组件本身就是纯函数，即传入指定props得到一定的Virtual DOM，整个过程都是可预测的。
> > ##### 具体办法
> > 拆分组件为子组件，对组件做更细粒度的控制。保持纯净状态，可以让方法或组件更加专注(focus)，体积更小(small)，更独立(independent)，更具有复用性(reusability)和可测试性(testability)。
> > 3. 运用PureRender，对变更做出最少的渲染
> > ##### 相关重要概念: PureRender
> > PureRender的Pure即是指满足纯函数的条件，即组件被相同的props和state渲染会得到相同的结果。在React中实现PureRender需要重新实现shouldComponentUpdate生命周期方法。shouldComponentUpdate是一个特别的方法，它接收需要更新的props和state，其本质是用来进行正确的组件渲染。当其返回false的时候，不再向下执行生命周期方法；当其返回true时，继续向下执行。组件在初始化过程中会渲染一个树状结构，当父节点props改变的时候，在理想情况下只需渲染一条链路上有关props改变的节点即可；但是，在默认情况下shouldComponentUpdate方法返回true,React会重新渲染所有的节点。
> > 有一些官方插件实现了对shouldComponentUpdate的重写，然后自己也可以做一些代码的优化来运用PureRender。
> > ##### 具体办法
> > > 1. 运用PureRender
> > > 使用官方插件react-addons-pure-render-mixin实现对shouldComponentUpdate的重写
> > > ```
> > > import React from 'react';
> > > import PureRenderMixin from 'react-addons-pure-render-mixin';
> > > 
> > > class App extends React.Component {
> > > constructor(props) {
> > >  super(props);
> > >  this.shouldComponentUpdate = PureRenderMixin.shouldComponentUpdate.bind(this);
> > > }
> > > render() {
> > >  return <div className={this.props.className}>foo</div>
> > > }
> > > }
> > > ```
> > > 它的原理是对object(包括props和state)做浅比较，即引用比较，非值比较。比如只用关注props中每一个是否全等(如果是prop是一个对象那就是只比较了地址，地址一样就算是一样了)，而不用深入比较。
> > > 2. 优化PureRender
> > > 避免无论如何都会触发shouldComponentUpdate返回true的代码写法。避免直接为prop设置字面量的数组和对象,就算每次传入的数组或对象的值没有变，但它们的地址也发生了变化。
> > > 如以下写法每次渲染时style都是新对象都会触发shouldComponentUpdate为true:
> > > `<Account style={color: 'black'} />`
> > > 改进办法：将字面量设置为一个引用:
> > > ```
> > > const defaultStyle = {};
> > > <Account style={this.props.style || defaultStyle} />
> > > ```
> > > 避免每次都绑定事件,如果这样绑定事件的话每次都要生成一个新的onChange属性的值:
> > > ```
> > > render() {
> > > return <input onChange={this.handleChange.bind(this)} />
> > > }
> > > ```
> > > 该尽量在构造函数内进行绑定，如果绑定需要传参那么应该考虑抽象子组件或改变现有数据结构:
> > > ```
> > > constructor(props) {
> > > super(props);
> > > this.handleChange = this.handleChange.bind(this);
> > > }
> > > handleChange() {
> > > ...
> > > }
> > > render() {
> > > return <input onChange={this.handleChange} />
> > > }
> > > ```
> > > 在设置子组件的时候要在父组件级别重写shouldComponentUpdate。
> > 4. 运用immutable
> > JavaScript中对象一般是可变的，因为使用引用赋值，新的对象的改变将影响原始对象。为了解决这个问题是使用深拷贝或者浅拷贝，但这样做又造成了CPU和内存的浪费。Immutable data很好地解决了这个问题。Immutable data就是一旦创建，就不能再更改的数据。对Immutable对象进行修改、添加或删除操作，都会返回一个新的Immutable对象。Immutable实现的原理是持久化的数据结构。即使用旧数据创建新数据时，保证新旧数据同时可用且不变。同时为了避免深拷贝带来的性能损耗，Immutable使用了结构共享(structural sharing),即如果对象树中一个节点发生变化，只修改这个节点和受它影响的父节点，其他节点则进行共享。
* ### 自动化测试
> jest 是 facebook 开源的，用来进行单元测试的框架，功能比较全面，测试、断言、覆盖率它都可以，另外还提供了快照功能。
> 对测试群众来说，从质量保证的角度出发，单元测试覆盖率100%是否就足够了呢？肯定不够啊！
> 结合实际的项目经验来看，jest的测试还可以根据产品的实际需求，做一些诸如：
> 点击某个页面元素后，需要在页面上显示新的区块，并且要加载指定的的css的测试;
> 点击某个link，需要跳转到指定的网站的测试;
> 等等
> 这些测试原本在UI自动化功能测试中也比较常见，这里我们都可以把它们挪到低层中去。所以具体的测试用例，在单元测试覆盖率超级高的前提下，我们测试的群众还可以跟研发结对完成。或者指导研发完成，要不干脆自己加上去算了。另外，产品的功能性测试完成的情况下，我们还需要考虑下非功能性的问题，例如兼容性、性能、安全性等。再加上测试金字塔的顶端之上，其实还有探索性测试的位置。产品的基本功能由单元测试保障了，剩下的时间，我们可以做更多的探索性测试了不是吗?总之，干掉UI自动化功能测试只是一个加速测试反馈周期、减少投入成本的尝试。软件的质量不仅仅是测试攻城狮的事情，而是整个团队的责任。坚持一些重要的编码实践，比如state less的组件、build security in等，也是提高质量的重要手段。