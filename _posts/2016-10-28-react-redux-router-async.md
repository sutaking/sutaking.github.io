---
layout: post
title: react+redux+router异步数据获取教程
date: 2016-10-28 15:32:24.000000000 +09:00
---

react的FLUX数据流一直搞不清楚，他不像`Angular`的双向数据绑定，做一个`model`获取数据，然后通过`controller`来管理`view上`的数据显示就可以了。单项数据流引入了太多的概念，`state`、`action`、`reducer`、`dispatch`。就算看的懂图，也不一定能coding出来。

不过我总算先搞定了`Redux`。
![redux img](https://github.com/sutaking/sutaking.github.io/blob/master/assets/images/redux.jpg?raw=true)

## keywords
-	**store**
- 	**reducer**
-	**action**
-	**dispatch**
-  **connect**
-  **router**
-  **middleware**
-  **thunk**

## Basic Usage
#### 1st 实现action方法
```javascript
export const addDeck = name => ({ type: 'ADD_DECK', data: name });
```

#### 2nd 根据action方法创建reducer方法
```javascript
export const showBack = (state, action) => {
  switch(action.type) {
    case 'SHOW_BACK':
      return action.data || false;
    default:
      return state || false;
  }
};
```
#### 3rd 根据reducer方法创建store
```javascript
const store = createStore(combineReducers(reducers));
```
`store.subscribe()`方法设置监听函数，一旦 State 发生变化，就自动执行这个函数。

`store.subscribe(listener);`
显然，只要把 View 的更新函数（对于 React 项目，就是组件的render方法或setState方法）放入listen，就会实现 View 的自动渲染。
store.subscribe方法返回一个函数，调用这个函数就可以解除监听。

```javascript
let unsubscribe = store.subscribe(() =>
  console.log(store.getState())
);

unsubscribe();
```

#### 4th 引入react-redux的<Provider>，导入store
```javascript
<Provider store={store}>
	{...}
</Provider>
```
#### 5th react组件中通过connect方法绑定store和dispatch。
```javascript
const mapStateToProps = (newTalks) => ({
    newTalks
});

const mapDispatchToProps = dispatch => ({
    testFunc: () => dispatch(updataTalkLists(1)),
    receiveData: () => dispatch(receiveData())
});

export default connect(mapStateToProps, mapDispatchToProps)(MainPage);
```
#### 6th this.props中直接调用action方法。
```javascript
this.props.receiveData
```
## With react-router
结合router使用时需要有2步。

#### 1st 绑定routing到reducer上
```javascript
import { syncHistoryWithStore, routerReducer } from 'react-router-redux';
import * as reducers from './redux/reducer';
reducers.routing = routerReducer;

const store = createStore(combineReducers(reducers));
```
#### 2nd 使用syncHistoryWithStore绑定store和browserHistory
```javascript
const history = syncHistoryWithStore(browserHistory, store);

		<Provider store={store}>
           <Router history={history}>
               {routes}
           </Router>
        </Provider>
```
## Async
类似 Express 或 Koa 框架中的中间件。它提供的是位于 action 被发起之后，到达 reducer 之前的扩展。
中间件的设计使用了非常多的函数式编程的思想，包括：高阶函数，复合函数，柯里化和ES6语法，源码仅仅20行左右。
项目中主要使用了三个中间件，分别解决不同的问题。

-	thunkMiddleware：处理异步Action
-	apiMiddleware：统一处理API请求。一般情况下，每个 API 请求都至少需要 dispatch 三个不同的 action（请求前、请求成功、请求失败），通过这个中间件可以很方便处理。
-	loggerMiddleware：开发环境调试使用，控制台输出应用state日志


实现action异步操作，必须要引入middleware。我这里用了`applyMiddleware(thunkMiddleware)`组件，也可以用其他的。

#### 1st 创建store是引入Middleware

```javascript
import thunkMiddleware from 'redux-thunk';
import { createStore, combineReducers, applyMiddleware } from 'redux';

const store = createStore(combineReducers(reducers), applyMiddleware(thunkMiddleware));
```
#### 2nd 创建一个可以执行dispacth的action
这也是中间件的作用所在。

```javascript
export const receiveData = data => ({ type: 'RECEIVE_DATA', data: data });

export const fetchData = () => {
  return dispatch => {
    fetch('/api/data')
      .then(res => res.json())
      .then(json => dispatch(receiveData(json)));
  };
};
```
#### 3rd 组件中对异步的store元素有相应的判断操作。
React组件会在store值发生变化时自动调用render()方法，更新异步数据。但是我们同样也需要处理异步数据没有返回或者请求失败的情况。否则渲染会失败，页面卡住。

```javascript
if(!data.newTalks) {
   return(<div/>);
}

```

## 相关知识
#### Store的实现
Store提供了3个方法

```javascript
import { createStore } from 'redux';
let { 
	subscribe, //监听store变化
	dispatch,  //调用action方法
	getState  //返回当前store
} = createStore(reducer);

```
下面是create方法的一个简单实现

```javascript
const createStore = (reducer) => {
  let state;
  let listeners = [];

  const getState = () => state;

  const dispatch = (action) => {
    state = reducer(state, action);
    listeners.forEach(listener => listener());
  };

  const subscribe = (listener) => {
    listeners.push(listener);
    return () => {
      listeners = listeners.filter(l => l !== listener);
    }
  };

  dispatch({});

  return { getState, dispatch, subscribe };
};
```

#### combineReducer的简单实现
```javascript
const combineReducers = reducers => {
  return (state = {}, action) => {
    return Object.keys(reducers).reduce(
      (nextState, key) => {
        nextState[key] = reducers[key](state[key], action);
        return nextState;
      },
      {} 
    );
  };
};
```

#### 中间件
-	createStore方法可以接受整个应用的初始状态作为参数，那样的话，applyMiddleware就是第三个参数了。
- 	中间件的次序有讲究，logger就一定要放在最后，否则输出结果会不正确。

```javascript
const store = createStore(
  reducer,
  initial_state,
  applyMiddleware(thunk, promise, logger)
);
```

applyMiddlewares的实现，它是将所有中间件组成一个数组，依次执行

```javascript
export default function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState, enhancer) => {
    var store = createStore(reducer, preloadedState, enhancer);
    var dispatch = store.dispatch;
    var chain = [];

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    };
    chain = middlewares.map(middleware => middleware(middlewareAPI));
    dispatch = compose(...chain)(store.dispatch);

    return {...store, dispatch}
  }
}
```
上面代码中，所有中间件被处理后得到一个数组保存在chain中。之后将chain传给compose，并将store.dispatch传给返回的函数。。可以看到，中间件内部（middlewareAPI）可以拿到getState和dispatch这两个方法。

那么在这里面做了什么呢？我们再看compose的实现：

```javascript
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
compose中的核心动作就是将传进来的所有函数倒序（reduceRight）进行如下处理：

```javascript
(composed, f) => f(composed)
```

我们知道Array.prototype.reduceRight是从右向左累计计算的，会将上一次的计算结果作为本次计算的输入。再看看applyMiddleware中的调用代码：

```javascript
dispatch = compose(...chain)(store.dispatch)
```
compose函数最终返回的函数被作为了dispatch函数，结合官方文档和代码，不难得出，中间件的定义形式为：

```javascript
function middleware({dispatch, getState}) {
    return function (next) {
        return function (action) {
            return next(action);
        }
    }
}

或  

middleware = (dispatch, getState) => next => action => {
    next(action);
}
```
也就是说，redux的中间件是一个函数，该函数接收dispatch和getState作为参数，返回一个以dispatch为参数的函数，这个函数的返回值是接收action为参数的函数（可以看做另一个dispatch函数）。在中间件链中，以dispatch为参数的函数的返回值将作为下一个中间件（准确的说应该是返回值）的参数，下一个中间件将它的返回值接着往下一个中间件传递，最终实现了store.dispatch在中间件间的传递。

#### redux-promise中间件
既然 Action Creator 可以返回函数，当然也可以返回其他值。另一种异步操作的解决方案，就是让 Action Creator 返回一个 Promise 对象。

-	写法一，返回值是一个 Promise 对象。

```javascript
const fetchPosts = 
  (dispatch, postTitle) => new Promise(function (resolve, reject) {
     dispatch(requestPosts(postTitle));
     return fetch(`/some/API/${postTitle}.json`)
       .then(response => {
         type: 'FETCH_POSTS',
         payload: response.json()
       });
});
```

- 写法二，Action 对象的payload属性是一个 Promise 对象。这需要从redux-actions模块引入createAction方法，并且写法也要变成下面这样。

```javascript
import { createAction } from 'redux-actions';

class AsyncApp extends Component {
  componentDidMount() {
    const { dispatch, selectedPost } = this.props
    // 发出同步 Action
    dispatch(requestPosts(selectedPost));
    // 发出异步 Action
    dispatch(createAction(
      'FETCH_POSTS', 
      fetch(`/some/API/${postTitle}.json`)
        .then(response => response.json())
    ));
  }
```
上面代码中，第二个dispatch方法发出的是异步 Action，只有等到操作结束，这个 Action 才会实际发出。注意，createAction的第二个参数必须是一个 Promise 对象。

redux-promise的源码

```javascript
export default function promiseMiddleware({ dispatch }) {
  return next => action => {
    if (!isFSA(action)) {
      return isPromise(action)
        ? action.then(dispatch)
        : next(action);
    }

    return isPromise(action.payload)
      ? action.payload.then(
          result => dispatch({ ...action, payload: result }),
          error => {
            dispatch({ ...action, payload: error, error: true });
            return Promise.reject(error);
          }
        )
      : next(action);
  };
}
```

从上面代码可以看出，如果 Action 本身是一个 Promise，它 resolve 以后的值应该是一个 Action 对象，会被dispatch方法送出（action.then(dispatch)），但 reject 以后不会有任何动作；如果 Action 对象的payload属性是一个 Promise 对象，那么无论 resolve 和 reject，dispatch方法都会发出 Action。

#### mapStateToProps()
-	mapStateToProps是一个函数。它的作用就是像它的名字那样，建立一个从（外部的）state对象到（UI 组件的）props对象的映射关系
-	mapStateToProps会订阅 Store，每当state更新的时候，就会自动执行，重新计算 UI 组件的参数，从而触发 UI 组件的重新渲染。
-	mapStateToProps的第一个参数总是state对象，还可以使用第二个参数，代表容器组件的props对象。
-	使用ownProps作为参数后，如果容器组件的参数发生变化，也会引发 UI 组件重新渲染。
- 	connect方法可以省略mapStateToProps参数，那样的话，UI 组件就不会订阅Store，就是说 Store 的更新不会引起 UI 组件的更新。

```javascript
// 容器组件的代码
//    <FilterLink filter="SHOW_ALL">
//      All
//    </FilterLink>

const mapStateToProps = (state, ownProps) => {
  return {
    active: ownProps.filter === state.visibilityFilter
  }
}
```
#### mapDispatchToProps()
mapDispatchToProps是connect函数的第二个参数，用来建立 UI 组件的参数到store.dispatch方法的映射。也就是说，它定义了哪些用户的操作应该当作 Action，传给 Store。它可以是一个函数，也可以是一个对象。

`mapDispatchToProps`作为函数，应该返回一个对象，该对象的每个键值对都是一个映射，定义了 UI 组件的参数怎样发出 Action。


```javascript
const mapDispatchToProps = (
  dispatch,
  ownProps
) => {
  return {
    onClick: () => {
      dispatch({
        type: 'SET_VISIBILITY_FILTER',
        filter: ownProps.filter
      });
    }
  };
}
```
如果`mapDispatchToProps`是一个对象，它的每个键名也是对应 UI 组件的同名参数，键值应该是一个函数，会被当作 `Action creator` ，返回的 `Action` 会由 `Redux` 自动发出。

```javascript
const mapDispatchToProps = {
  onClick: (filter) => {
    type: 'SET_VISIBILITY_FILTER',
    filter: filter
  };
}
```

#### `<Provider>` 组件
React-Redux 提供Provider组件，可以让容器组件拿到state，它的原理是React组件的context属性，请看源码。

```javascript
class Provider extends Component {
  getChildContext() {
    return {
      store: this.props.store
    };
  }
  render() {
    return this.props.children;
  }
}

Provider.childContextTypes = {
  store: React.PropTypes.object
}
```
上面代码中，store放在了上下文对象context上面。然后，子组件就可以从context拿到store，代码大致如下。

```javascript
class VisibleTodoList extends Component {
  componentDidMount() {
    const { store } = this.context;
    this.unsubscribe = store.subscribe(() =>
      this.forceUpdate()
    );
  }

  render() {
    const props = this.props;
    const { store } = this.context;
    const state = store.getState();
    // ...
  }
}

VisibleTodoList.contextTypes = {
  store: React.PropTypes.object
}
```

#### redux-thunk
我们知道，异步调用什么时候返回前端是无法控制的。对于redux这条严密的数据流来说，如何才能做到异步呢。redux-thunk的基本思想就是通过函数来封装异步请求，也就是说在actionCreater中返回一个函数，在这个函数中进行异步调用。我们已经知道，redux中间件只关注dispatch函数的传递，而且redux也不关心dispatch函数的返回值，所以只需要让redux认识这个函数就可以了。
看了一下redux-thunk的源码：

```javascript
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
这段代码跟上面我们看到的中间件没有太大的差别，唯一一点就是对action做了一下如下判断：

```javascript
if (typeof action === 'function') {
   return action(dispatch, getState, extraArgument);
}
```
也就是说，如果发现actionCreater传过来的action是一个函数的话，会执行一下这个函数，并以这个函数的返回值作为返回值。前面已经说过，redux对dispatch函数的返回值不是很关心，因此此处也就无所谓了。

这样的话，在我们的actionCreater中，我们就可以做任何的异步调用了，并且返回任何值也无所谓，所以我们可以使用promise了：

```javascript
function actionCreate() {
    return function (dispatch, getState) {
        // 返回的函数体内自由实现。。。
        Ajax.fetch({xxx}).then(function (json) {
            dispatch(json);
        })
    }
}
```
最后还需要注意一点，由于中间件只关心dispatch的传递，并不限制你做其他的事情，因此我们最好将redux-thunk放到中间件列表的首位，防止其他中间件中返回异步请求。