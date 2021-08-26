整个Redux的核心就在于createStore这个函数，这也是我们分析Redux的入口点

关于createStore，主要的就是接受三个参数，分别是reducer, preloadedState, enhancer

其中reducer用来表示对于数据响应的操作，preloadedState用来表示传入的初始state，enhancer则用来表示对于dispatch方法的增强

实现一个最基本的createStore（暂时不考虑enhancer），如下所示，返回getState,dispatch,subscribe三个方法

```
function createStore (reducer, preloadedState, enhancer) {
  // reducer 类型判断 
  if (typeof reducer !== 'function') throw new Error('redcuer必须是函数');

  // 状态
  var currentState = preloadedState;
  // 订阅者
  var currentListeners = [];
  // 获取状态
  function getState () {
    return currentState;
  }
  // 用于触发action的方法
  function dispatch (action) {
    // 判断action是否是一个对象
    if (!isPlainObject(action)) throw new Error('action必须是一个对象');
    // 判断action中的type属性是否存在
    if (typeof action.type === 'undefined') throw new Error('action对象中必须有type属性');
    // 调用reducer函数 处理状态
    currentState = reducer(currentState, action);
    // 调用订阅者 通知订阅者状态发生了改变
    for (var i = 0; i < currentListeners.length; i++) {
      var listener = currentListeners[i];
      listener();
    }
  }
  // 订阅状态的改变
  function subscribe (listener) {
    currentListeners.push(listener);
  }

  // 默认调用一次dispatch方法 存储初始状态(通过reducer函数传递的默认状态)
  dispatch({type: 'initAction'})

  return {
    getState,
    dispatch,
    subscribe
  }
}
```

1、getState是用来获取当前最新的state，它的实现很简单，就是返回currentState

2、dispatch用来派发具体的action给对应的reducer处理，并返回处理结果给currentState，并通知所有订阅了该store的方法进行更新，具体实现就是一个循环调用currentListeners这个订阅数组里的listener函数。

3、subscribe用来添加listener函数的，每一次调用就会添加一个监听函数到currentListeners这个数组当中

这个时候我们就完成了最简单的Redux方法了

下面我们来考虑加入enhancer方法对dispatch方法进行增强，同样是在createStore这个函数体当中，我们加入如下代码

```
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('enhancer必须是函数')
    }
    return enhancer(createStore)(reducer, preloadedState);
  }
```

我们之前已经分析过了，createStore通过reducer, preloadedState就可以创建出一套可用的redux，所以在执行enhancer函数之后就可以返回一个完整的store，

同时会把对应的dispatch方法做增强然后替换，一般来说调用方法如下

```
      function enhancer (createStore) {
        return function (reducer, preloadedState) {
          var store = createStore(reducer, preloadedState);
          var dispatch = store.dispatch;
          function _dispatch (action) {
            if (typeof action === 'function') {
              return action(dispatch)
            }
            dispatch(action);
          }
          return {
            ...store,
            dispatch: _dispatch
          }
        }
      }
```

通过新定义的_dispatch 方法来替代dispatch方法，在该方法当中，对传入的action做一个判断，如果是函数就执行action，否则就用基本store里的dispatch执行

那么在解决了enhancer方法的调用问题之后，我们来实现一下applyMiddleware这个内置的中间件方法，如下所示其本质就是使用enhancer方法增强我们的dispatch方法，


```
function applyMiddleware (...middlewares) {
  return function (createStore) {
    return function (reducer, preloadedState) {
      // 创建 store
      var store = createStore(reducer, preloadedState);
      // 阉割版的 store
      var middlewareAPI = {
        getState: store.getState,
        dispatch: store.dispatch
      }
      // 调用中间件的第一层函数 传递阉割版的store对象
      var chain = middlewares.map(middleware => middleware(middlewareAPI));
      var dispatch = compose(...chain)(store.dispatch);
      return {
        ...store,
        dispatch
      }
    }
  }
}


```

在applyMiddleware当中，首先返回的就是一个enhancer函数模板

然后当createStore执行enhancer(createStore)(reducer, preloadedState)时，就会执行这个函数

在这之中首先来创建一个最简易版的store，然后开始执行以下两行

```
      // 调用中间件的第一层函数 传递阉割版的store对象
      var chain = middlewares.map(middleware => middleware(middlewareAPI));
      var dispatch = compose(...chain)(store.dispatch);
```


这里要配合常见的middlewares函数进行说明，例如logger

```
function logger (store) {
  return function (next) {
    return function (action) {
      console.log('logger');
      next(action)
    }
  }
}
```

这里的middleware(middlewareAPI)就是在调用logger (store)，然后chain就是存放每一个middleware返回的函数的数组，

然后再调用compose(...chain)(store.dispatch)方法完成一个dispatch方法的替换，对于compose方法如下所示

```
function compose () {
  var funcs = [...arguments];
  return function (dispatch) {
    for (var i = funcs.length - 1; i >= 0; i--) {
      dispatch = funcs[i](dispatch);
    }
    return dispatch;
  }
}
```

对于传入的函数数组也就是chain用arguments拿到，然后再执行以下的函数，进行一个for循环，注意此处的for循环是倒序进行，

执行流程是对于chain数组的最后一个函数，执行如下部分并把store的dispatch传入，也就是说此时的next为store.dispatch，并把返回赋值给dispatch

```
  return function (next) {
    return function (action) {
      console.log('logger');
      next(action)
    }
  }
```

然后针对倒数第二个函数，还是传入一个dispatch，注意此时的dispatch也就是next就是刚刚chain最后一个函数执行后返回的函数，最后返回dispatch用来替代原本的store.dispatch

那么当我们调用dispatch的时候，就会按照我们注册的顺序执行下来






