# React学习之深入Redux应用框架

Redux作为大型React应用状态管理最常用的工具。它是一个应用数据流框架，与Flux框架类似。它是零依赖的，可以配合其他框架或者类库一起使用。虽然在平时的工作中很多次的用到了它，但是一直没有对其原理进行研究。最近看了一下源码，下面是我自己的一些简单认识。
1. createStore
结合使用场景我们首先来看一下createStore方法。
```
// 这是我们平常使用时创建store
const store = createStore(reducers, state, enhance);
```
以下源码为去除异常校验后的源码，
```

export default function createStore(reducer, preloadedState, enhancer) {

// 如果有传入合法的enhance，则通过enhancer再调用一次createStore
 if (typeof enhancer !== 'undefined') {
   if (typeof enhancer !== 'function') {
     throw new Error('Expected the enhancer to be a function.')
   }
   return enhancer(createStore)(reducer, preloadedState) // 这里涉及到中间件，后面介绍applyMiddleware时在具体介绍
 }

 let currentReducer = reducer     //把 reducer 赋值给 currentReducer
 let currentState = preloadedState   //把 preloadedState 赋值给 currentState
 let currentListeners = []     //初始化监听函数列表
 let nextListeners = currentListeners   //监听列表的一个引用
 let isDispatching = false  //是否正在dispatch

 function ensureCanMutateNextListeners() {}

 function getState() {}

 function subscribe(listener) {}
 
 function dispatch(action) {}

 function replaceReducer(nextReducer) {}
 
// 在 creatorStore 内部没有看到此方法的调用，就不讲了
 function observable() {}
 //初始化 store 里的 state tree
 dispatch({ type: ActionTypes.INIT })

 return {
   dispatch,
   subscribe,
   getState,
   replaceReducer,
   [$$observable]: observable
 }
}
```
我们可以看到creatorStore方法除了返回我们常用的方法外，还做了一次初始化过程dispatch({ type: ActionTypes.INIT });那么dispatch干了什么事情呢？
```
/**
  * dispath action。这是触发 state 变化的惟一途径。
  * @param {Object} 一个普通(plain)的对象，对象当中必须有 type 属性
  * @returns {Object} 返回 dispatch 的 action
  */
 function dispatch(action) {
 // 判断 dispahch 正在运行，Reducer在处理的时候又要执行 dispatch
   if (isDispatching) {
     throw new Error('Reducers may not dispatch actions.')
   }

   try {
     //标记 dispatch 正在运行
     isDispatching = true
     //执行当前 Reducer 函数返回新的 state
     currentState = currentReducer(currentState, action)
   } finally {
     isDispatching = false
   }
   const listeners = (currentListeners = nextListeners)
   //遍历所有的监听函数
   for (let i = 0; i < listeners.length; i++) {
     const listener = listeners[i]
     listener() // 执行每一个监听函数
   }
   return action
 }
```
这里dispatch主要做了二件事情
* 通过reducer更新state
* 执行所有的监听函数，通知状态的变更
那么reducer是怎么改变state的呢？这就涉及到下面的combineReducers了。
2. combineReducers
回到上面创建store的参数reducers，
```
// 两个reducer
const todos = (state = INIT.todos, action) => {
  // ....
};
const filterStatus = (state = INIT.filterStatus, action) => {
  // ...
};

const reducers = combineReducers({
  todos,
  filterStatus
});
// 这是我们平常使用时创建store
const store = createStore(reducers, state, enhance);
```
下面我们来看combineReducers做了什么
```export default function combineReducers(reducers) {
   // 第一次筛选，参数reducers为Object
  // 筛选掉reducers中不是function的键值对
  const reducerKeys = Object.keys(reducers)
  const finalReducers = {}
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)
  
  // 二次筛选，判断reducer中传入的值是否合法（!== undefined）
  // 获取筛选完之后的所有key
  let shapeAssertionError
  try {
    assertReducerShape(finalReducers)
  } catch (e) {
    shapeAssertionError = e
  }

  return function combination(state = {}, action) {
    let hasChanged = false
    const nextState = {}
    // 遍历所有的key和reducer，分别将reducer对应的key所代表的state，代入到reducer中进行函数调用
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      // 这里就是reducer function的名称和要和state同名的原因，传说中的黑魔法
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      // 将reducer返回的值填入nextState
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    // 发生改变了返回新的nextState，否则返回原先的state
    return hasChanged ? nextState : state
  }
}
```
* 这里 reducer(previousStateForKey, action)执行的就是我们上面定义的todos和filterStatus方法。通过这二个reducer改变state值。
* 以前我一直很奇怪我们的reducer里的state是怎么做到取当前reducer对应的数据。看到const previousStateForKey = state[key]这里我就明白了。
* 这里还有一个疑问点就是combineReducers的嵌套，最开始也我不明白，看了源码才知道combineReducers（）=> combination(state = {}, action)，这里combineReducers返回的combination也是接受(state = {}, action)也就是一个reducer所以可以正常嵌套。
看到这初始化流程已经走完了。这个过程我们认识了dispatch和combineReducers;接下来我们来看一下我们自己要怎么更新数据。

用户更新数据时，是通过createStore后暴露出来的dispatch方法来触发的。dispatch 方法，是 store 对象提供的更改 currentState 这个闭包变量的唯一建议途径（注意这里是唯一建议途径，不是唯一途径，因为通过getState获取到的是state的引用，所以是可以直接修改的。但是这样就不能更新视图了）。
正常情况下我们只需要像下面这样
```
//action creator
var addTodo = function(text){
    return {
        type: 'add_todo',
        text: text
    };
};
function TodoReducer(state = [], action){
    switch (action.type) {
        case 'add_todo':
            return state.concat(action.text);
        default:
            return state;
    }
};

// 通过 store.dispatch(action) 来达到修改 state 的目的
// 注意: 在redux里,唯一能够修改state的方法,就是通过 store.dispatch(action)
store.dispatch({type: 'add_todo', text: '读书'});// 或者下面这样
// store.dispatch(addTodo('读书'));
```
也就是说dispatch接受一个包含type的对象。框架为我们提供了一个创建Action的方法bindActionCreators。
3. bindActionCreators
下面来看下源码
```
// 核心代码，并通过apply将this绑定起来
function bindActionCreator(actionCreator, dispatch) {
  return function() {
    return dispatch(actionCreator.apply(this, arguments))
  }
}

export default function bindActionCreators(actionCreators, dispatch) {
// 如果actionCreators是一个函数，则说明只有一个actionCreator，就直接调用bindActionCreator
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }
  // 遍历对象，然后对每个遍历项的 actionCreator 生成函数，将函数按照原来的 key 值放到一个对象中，最后返回这个对象
  const keys = Object.keys(actionCreators)
  const boundActionCreators = {}
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i]
    const actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  return boundActionCreators
}
```
bindActionCreators的作用就是使用dispatch把action creator包裹起来，这样我们就可以直接调用他们了。这个在平常开发中不常用。
4. applyMiddleware
最后我们回头来看一下之前调到的中间件，
```
import thunkMiddleware from 'redux-thunk';
// 两个reducer
const todos = (state = INIT.todos, action) => {
  // ....
};
const filterStatus = (state = INIT.filterStatus, action) => {
  // ...
};

const reducers = combineReducers({
  todos,
  filterStatus
});
// 这是我们平常使用时创建store
const store = createStore(reducers, state, applyMiddleware(thunkMiddleware));
```
为了下文好理解这个放一下redux-thunk的源码
```
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    return next(action);
  };
}
const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;
export default thunk;
```
可以看出thunk返回了一个接受({ dispatch, getState })为参数的函数
下面我们来看一下applyMiddleware源码分析
```
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    // 每个 middleware 都以 middlewareAPI 作为参数进行注入，返回一个新的链。此时的返回值相当于调用 thunkMiddleware 返回的函数： (next) => (action) => {} ，接收一个next作为其参数
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 并将链代入进 compose 组成一个函数的调用链
    // compose(...chain) 返回形如(...args) => f(g(h(...args)))，f/g/h都是chain中的函数对象。
    // 在目前只有 thunkMiddleware 作为 middlewares 参数的情况下，将返回 (next) => (action) => {}
    // 之后以 store.dispatch 作为参数进行注入注意这里这里的store.dispatch是没有被修改的dispatch他被传给了next；
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}

// 定义一个代码组合的方法
// 传入一些function作为参数，返回其链式调用的形态。例如，
// compose(f, g, h) 最终返回 (...args) => f(g(h(...args)))
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  } else {
    const last = funcs[funcs.length - 1]
    const rest = funcs.slice(0, -1)
    return (...args) => rest.reduceRight((composed, f) => f(composed), last(...args))
  }
}
```
也就是一个三级柯里化的函数，我们从头来分析一下这个过程
```
// createStore.js
if (typeof enhancer !== 'undefined') {
  if (typeof enhancer !== 'function') {
    throw new Error('Expected the enhancer to be a function.')
  }
  return enhancer(createStore)(reducer, preloadedState)
}
```
也就是说，会变成这样
```
applyMiddleware(thunkMiddleware)(createStore)(reducer, preloadedState)
```
* applyMiddleware(thunkMiddleware)
> applyMiddleware接收thunkMiddleware作为参数，返回形如(createStore) => (...args) => {}的函数。
* applyMiddleware(thunkMiddleware)(createStore)
> 以 createStore 作为参数，调用上一步返回的函数(...args) => {}
* applyMiddleware(thunkMiddleware)(createStore)(reducer, preloadedState)
> 以（reducer, preloadedState）为参数进行调用。 在这个函数内部，thunkMiddleware被调用，其作用是监测type是function的action。
> 因此，如果dispatch的action返回的是一个function，则证明是中间件，则将(dispatch, getState)作为参数代入其中，进行action 内部下一步的操作。否则的话，认为只是一个普通的action，将通过next(也就是dispatch)进一步分发。
---
也就是说，applyMiddleware(thunkMiddleware)作为enhance，最终起了这样的作用：

对dispatch调用的action进行检查，如果action在第一次调用之后返回的是function，则将(dispatch, getState)作为参数注入到action返回的方法中，否则就正常对action进行分发，这样一来我们的中间件就完成了。
> 因此，当action内部需要获取state，或者需要进行异步操作，在操作完成之后进行事件调用分发的话，我们就可以让action 返回一个以(dispatch, getState)为参数的function而不是通常的Object，enhance就会对其进行检测以便正确的处理。

到此redux源码的主要部分学习结束。