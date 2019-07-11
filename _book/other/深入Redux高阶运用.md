# React学习之深入Redux高阶运用

在Redux架构中，reducer是一个纯函数，它的职责是根据previousState和action计算出新的state。在复杂应用中，Redux提供的combineReducers让我们可以把顶层的reducer拆分成多个小的reducer，分别独立地操作state树的不同部分。而在一个应用中，很多小粒度的reducer往往有很多重复的逻辑，那么对于这些reducer，如何抽取公共逻辑，减少代码冗余呢？这种情况下，使用高阶reducer是一种较好的解决方案
1. reducer复用
我们将顶层的reduce拆分成多个小的reducer，肯定会碰到reducer复用问题。例如有A和B两个模块，它们的UI部分相似，此时可以通过配置不同的props来区别它们。那么这种情况下，A和B模块能不能共用一个reducer呢？答案是否定的。我们先来看一个简单reducer：
```
const LOAD_DATA = 'LOAD_DATA';
const initialState = { ... };

function loadData() {
    return {
        type: LOAD_DATA,
        ...
    };
}

function reducer(state = initialState, action) {
    switch(action.type) {
        case LOAD_DATA:
            return {
                ...state,
                data: action.payload
            };
        default:
            return state;
    }
}
```
如果我们将这个reducer绑定到A和B两个不同模块，造成的问题将会是，当A模块调用loadData来分发相应的action时，A和B的reducer都会处理这个action，然后A和B的内容就完全一致了。

这里我们必需意识到，在一个应用中，不同模块间的actionType必须是全局唯一的。

因此，要解决actionType唯一的问题，还有一个方法就是通过添加前缀的方式来做到：
```
function generateReducer(prefix, state) {
    const LOAD_DATA = prefix + 'LOAD_DATA';
    
    const initialState = { ...state, ...};
    
    return function reducer(state = initialState, action) {
        switch(action.type) {
            case LOAD_DATA:
                return {
                    ...state,
                    data: action.payload
                };
            default:
                return state;
        }
    }
}
```
这样只要A和B模块分别调用generateReducer来生成相应的reducer，就能解决reducer复用的问题了。而对于prefix，我们可以根据自己的项目结构来决定，例如${页面名称}_${模块名称}。只要能够保证全局唯一性，就可以写成一种前缀。
2. reducer增强
除了解决复用问题，高阶reducer的另一个重要作用就是对原始的reducer进行增强。redux-undo就是典型的利用高阶reducer来增强reducer的例子，它主要作用是使任意reducer变成可以执行撤销和重做的全新reducer。我们来看看它的核心代码实现：
```
function undoable(reducer) {
    const initialState = {
        // 记录过去的state
        past: [],
        // 以一个空的action调用reducer来产生当前值的初始值
        present: reducer(undefined, {}),
        // 记录后续的state
        future: []
    };
    
    return function(state = initialState, action) {
        const { past, present, future } = state;
        
        switch(action.type) {
            case '@@redux-undo/UNDO':
                const previous = past[past.length - 1];
                const newPast = past.slice(0, past.length - 1);
                
                return {
                    past: newPast,
                    present: previous,
                    future: [ present, ...future ]
                };
            case '@@redux-undo/REDO':
                const next = future[0];
                const newFuture = future.slice(1);
                
                return {
                    past: [ ...past, present ],
                    present: next,
                    future: newFuture
                };
            default:
                // 将其他action委托给原始的reducer处理
                const newPresent = reducer(present, action);
                
                if(present === newPresent) {
                    return state;
                }
                
                return {
                    past: [ ...past, present ],
                    present: newPresent,
                    future: []
                };
        }
    };
}
```
有了这高阶reducer，就可以对任意一个reducer进行封装：
```
import { createStore } from 'redux';

function todos(state = [], action) {
    switch(action.type) {
        case: 'ADD_TODO':
        // ...
    }
}

const undoableTodos = undoable(todos);
const store = createStore(undoableTodos);

store.dispatch({
    type: 'ADD_TODO',
    text: 'Use Redux'
});

store.dispatch({
    type: 'ADD_TODO',
    text: 'Implement Undo'
});

store.dispatch({
    type: '@@redux-undo/UNDO'
});
```
查看高阶reducer undoable的实现代码可以发现，高阶reducer主要通过下面3点来增强reducer：
能够处理额外的action;
能够维护更多的state;
将不能处理的action委托给原始reducer处理。