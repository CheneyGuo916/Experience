# React学习之解读React源码

首先理解ReactElement和ReactClass的概念。想要更好的利用react的虚拟DOM，diff算法的优势，我们需要正确的优化、组织react页面。
* ### 理解ReactElement和ReactClass的概念
> #### ReactElement
> 一个描述DOM节点或component实例的字面级对象。它包含一些信息，包括组件类型type和属性props。就像一个描述DOM节点的元素(虚拟节点)。它们可以被创建通过React.createElement方法或jsx写法
> 分为DOM Element和Component Elements两类：
> > * DOM Elements
> > 当节点的type属性为字符串时，它代表是普通的节点，如div,span
> > ```
> > { 
> > type: 'button', 
> > props: { 
> >  className: 'button button-blue', 
> >  children: { 
> >    type: 'b', 
> >    props: { 
> >      children: 'OK!' 
> >    } 
> >  } 
> > } 
> > }  
> > ```
> > * Component Elements
> > 当节点的type属性为一个函数或一个类时，它代表自定义的节点
> > ```
> > class Button extends React.Component { 
> > render() { 
> >  const { children, color } = this.props; 
> >  return { 
> >    type: 'button', 
> >    props: { 
> >      className: 'button button-' + color, 
> >      children: { 
> >        type: 'b', 
> >        props: { 
> >         children: children 
> >        } 
> >      } 
> >    } 
> >  }; 
> > } 
> > } 
> > 
> > // Component Elements 
> > { 
> > type: Button, 
> > props: { 
> >  color: 'blue', 
> >  children: 'OK!' 
> > } 
> > }  
> > ```
> #### ReactElement
> ReactClass是平时我们写的Component组件(类或函数)，例如上面的Button类。ReactClass实例化后调用render方法可返回DOM Element。
> #### react渲染过程
> 过程理解:
> ```
> // element是 Component Elements 
> ReactDOM.render({ 
> type: Form, 
> props: { 
>  isSubmitted: false, 
>  buttonText: 'OK!' 
> } 
> }, document.getElementById('root'));
> ```
> > 1. 调用React.render方法，将我们的element根虚拟节点渲染到container元素中。element可以是一个字符串文本元素，也可以是如上介绍的ReactElement(分为DOM Elements, Component Elements)。
> > 2. 根据element的类型不同，分别实例化ReactDOMTextComponent, ReactDOMComponent, ReactCompositeComponent类。这些类用来管理ReactElement,负责将不同的ReactElement转化成DOM(mountComponent方法),负责更新DOM(receiveComponent方法，updateComponent方法, 如下会介绍)等。
> > 3. ReactCompositeComponent实例调用mountComponent方法后内部调用render方法，返回了DOM Elements。
> #### react更新机制
> 每个类型的元素都要处理好自己的更新：
> > 1. 自定义元素的更新，主要是更新render出的节点，做甩手掌柜交给render出的节点的对应component去管理更新。
> > 2. text节点的更新很简单，直接更新文案。
> > 3. 浏览器基本元素的更新，分为两块：
> > > * 先是更新属性，对比出前后属性的不同，局部更新。并且处理特殊属性，比如事件绑定。
> > > * 然后是子节点的更新，子节点更新主要是找出差异对象，找差异对象的时候也会使用上面的shouldUpdateReactComponent来判断，如果是可以直接更新的就会递归调用子节点的更新,这样也会递归查找差异对象。不可直接更新的删除之前的对象或添加新的对象。之后根据差异对象操作dom元素(位置变动，删除，添加等)。
> #### 第一步：调用this.setState
> ```
> ReactClass.prototype.setState = function(newState) { 
>  //this._reactInternalInstance是ReactCompositeComponent的实例 
>  this._reactInternalInstance.receiveComponent(null, newState); 
> }  
> ```
> #### 第二步：调用内部receiveComponent方法
> 这里主要分三种情况，文本元素，基本元素，自定义元素。
> 自定义元素:
> receiveComponent方法源码:
> ```
> // receiveComponent方法 
> ReactCompositeComponent.prototype.receiveComponent = function(nextElement, transaction, nextContext) { 
>  var prevElement = this._currentElement; 
>  var prevContext = this._context; 
> 
>  this._pendingElement = null; 
> 
>  this.updateComponent( 
>    transaction, 
>    prevElement, 
>    nextElement, 
>    prevContext, 
>    nextContext 
>  ); 
> 
> }  
> ```
> updateComponent方法源码
> ```
> // updateComponent方法 
> ReactCompositeComponent.prototype.updateComponent = function( 
> transaction, 
> prevParentElement, 
> nextParentElement, 
> prevUnmaskedContext, 
> nextUnmaskedContext 
> ) { 
> // 简写..... 
>  
>  // 不是state更新而是props更新 
>  if (prevParentElement !== nextParentElement) { 
>    willReceive = true; 
>  } 
> 
>  if (willReceive && inst.componentWillReceiveProps) { 
>      // 调用生命周期componentWillReceiveProps方法 
>  } 
>   
>  // 是否更新元素 
>  if (inst.shouldComponentUpdate) { 
>      // 如果提供shouldComponentUpdate方法 
>      shouldUpdate = inst.shouldComponentUpdate(nextProps, nextState, nextContext); 
>  } else { 
>      if (this._compositeType === CompositeTypes.PureClass) { 
>        // 如果是PureClass，浅层对比props和state 
>        shouldUpdate = 
>          !shallowEqual(prevProps, nextProps) || 
>          !shallowEqual(inst.state, nextState); 
>      } 
>  } 
>   
>  if (shouldUpdate) { 
>    // 更新元素 
>    this._performComponentUpdate( 
>      nextParentElement, 
>      nextProps, 
>      nextState, 
>      nextContext, 
>      transaction, 
>      nextUnmaskedContext 
>    ); 
>  } else { 
>    // 不更新元素，但仍然设置props和state 
>    this._currentElement = nextParentElement; 
>    this._context = nextUnmaskedContext; 
>    inst.props = nextProps; 
>    inst.state = nextState; 
>    inst.context = nextContext; 
>  } 
>      
> // ....... 
> 
> }  
> ```
> 内部_performComponentUpdate方法源码
> ```
> function shouldUpdateReactComponent(prevElement, nextElement){ 
> var prevEmpty = prevElement === null || prevElement === false; 
> var nextEmpty = nextElement === null || nextElement === false; 
> if (prevEmpty || nextEmpty) { 
>  return prevEmpty === nextEmpty; 
> } 
> 
> var prevType = typeof prevElement; 
> var nextType = typeof nextElement; 
> 
> if (prevType === 'string' || prevType === 'number') { 
>  // 如果先前的ReactElement对象类型是字符串或数字，新的ReactElement对象类型也是字符串或数字，
>  return (nextType === 'string' || nextType === 'number'); 
> } else { 
>    // 如果先前的ReactElement对象类型是对象，新的ReactElement对象类型也是对象，并且标签类型和key值相同，则需要更新 
>   return ( 
>     nextType === 'object' && 
>     prevElement.type === nextElement.type && 
>     prevElement.key === nextElement.key 
>   ); 
> } 
> }
> ```
> 基本元素
> receiveComponent方法源码
> ```
> ReactDOMComponent.prototype.receiveComponent = function(nextElement, transaction, context) { 
> var prevElement = this._currentElement; 
>   this._currentElement = nextElement; 
>  this.updateComponent(transaction, prevElement, nextElement, context); 
> }  
> ```
> updateComponent方法源码
> ```
> ReactDOMComponent.prototype.updateComponent = function(transaction, prevElement, nextElement, context) { 
>  // 略..... 
>  //需要单独的更新属性 
>  this._updateDOMProperties(lastProps, nextProps, transaction, isCustomComponentTag); 
>  //再更新子节点 
>  this._updateDOMChildren( 
>    lastProps, 
>    nextProps, 
>    transaction, 
>    context 
>  ); 
> 
>  // ...... 
> }  
> ```
> this._updateDOMChildren方法内部调用diff算法。
> #### react Diff算法
> diff算法源码
> ```
> _updateChildren: function(nextNestedChildrenElements, transaction, context) { 
>  var prevChildren = this._renderedChildren; 
>  var removedNodes = {}; 
>  var mountImages = []; 
>   
>  // 获取新的子元素数组 
>  var nextChildren = this._reconcilerUpdateChildren( 
>    prevChildren, 
>    nextNestedChildrenElements, 
>    mountImages, 
>    removedNodes, 
>    transaction, 
>    context 
>  ); 
>   
>  if (!nextChildren && !prevChildren) { 
>    return; 
>  } 
>   
>  var updates = null; 
>  var name; 
>  var nextIndex = 0; 
>  var lastIndex = 0; 
>  var nextMountIndex = 0; 
>  var lastPlacedNode = null; 
> 
>  for (name in nextChildren) { 
>    if (!nextChildren.hasOwnProperty(name)) { 
>      continue; 
>    } 
>    var prevChild = prevChildren && prevChildren[name]; 
>    var nextChild = nextChildren[name]; 
>    if (prevChild === nextChild) { 
>        // 同一个引用，说明是使用的同一个component,所以我们需要做移动的操作 
>        // 移动已有的子节点 
>        // NOTICE：这里根据nextIndex, lastIndex决定是否移动 
>      updates = enqueue( 
>        updates, 
>        this.moveChild(prevChild, lastPlacedNode, nextIndex, lastIndex) 
>      ); 
>       
>      // 更新lastIndex 
>      lastIndex = Math.max(prevChild._mountIndex, lastIndex); 
>      // 更新component的.mountIndex属性 
>      prevChild._mountIndex = nextIndex; 
>       
>    } else { 
>      if (prevChild) { 
>        // 更新lastIndex 
>        lastIndex = Math.max(prevChild._mountIndex, lastIndex); 
>      } 
>       
>      // 添加新的子节点在指定的位置上 
>      updates = enqueue( 
>        updates, 
>        this._mountChildAtIndex( 
>          nextChild, 
>          mountImages[nextMountIndex], 
>          lastPlacedNode, 
>          nextIndex, 
>          transaction, 
>          context 
>        ) 
>      ); 
>       
>       
>      nextMountIndex++; 
>    } 
>     
>    // 更新nextIndex 
>    nextIndex++; 
>    lastPlacedNode = ReactReconciler.getHostNode(nextChild); 
>  } 
>   
>  // 移除掉不存在的旧子节点，和旧子节点和新子节点不同的旧子节点 
>  for (name in removedNodes) { 
>    if (removedNodes.hasOwnProperty(name)) { 
>      updates = enqueue( 
>        updates, 
>        this._unmountChild(prevChildren[name], removedNodes[name]) 
>      ); 
>    } 
>  } 
> }  
> ```
> #### react的优点与总结
> #### 优点：
> > * 虚拟节点。在UI方面，不需要立刻更新视图，而是生成虚拟DOM后统一渲染。
> > * 组件机制。各个组件独立管理,层层嵌套，互不影响，react内部实现的渲染功能。
> > * 差异算法。根据基本元素的key值，判断是否递归更新子节点，还是删除旧节点，添加新节点。
> #### 总结：
> 想要更好的利用react的虚拟DOM，diff算法的优势，我们需要正确的优化、组织react页面。例如将一个页面render的ReactElement节点分解成多个组件。在需要优化的组件手动添加 shouldComponentUpdate 来避免不需要的 re-render。