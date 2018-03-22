---
title: test
---

# react 状态管理+异步 思考


[对使用Redux和Redux-saga管理状态的思考 - 莫凡的前端专栏 - SegmentFault 思否](https://segmentfault.com/a/1190000011523186)值得一看，深度剖析
[聊聊redux-saga - 贰年人折木 - SegmentFault 思否](https://segmentfault.com/a/1190000011104199)有一个登录登出的例子
[Redux异步方案选型 - kpaxqin - SegmentFault 思否](https://segmentfault.com/a/1190000007248878)作者的视野比较高，从模板代码、乐观更新、错误处理、竞态等多方面进行选型分析


## 渐进式地选择数据流工具
###### Context
我需要一个全局数据源且其他组件可以直接获取/改变全局数据源中的数据

###### Redux
1. 我需要一个全局数据源且其他组件可以直接获取/改变全局数据源中的数据
2. 我需要全程跟踪/管理 action 的分发过程/顺序

###### redux-thunk
[Redux异步控制](bear://x-callback-url/open-note?id=19F38F31-215E-4803-8471-5AD3ABD7D0A2-321-0000635354743DFF)
1. 我需要一个全局数据源且其他组件可以直接获取/改变全局数据源中的数据
2. 我需要全程跟踪/管理 action 的分发过程/顺序
3. 我需要组件对同步或异步的 action 无感，调用异步 action 时不需要显式地传入 dispatch

redux-thunk 的主要思想是扩展 action，使得 action 从一个对象变成一个函数。

###### redux-saga
1. 我需要一个全局数据源且其他组件可以直接获取/改变全局数据源中的数据
2. 我需要全程跟踪/管理 action 的分发过程/顺序
3. 我需要组件对同步或异步的 action 无感，调用异步 action 时不需要显式地传入 dispatch
4. 我需要声明式地来表述复杂异步数据流（如长流程表单，请求失败后重试等），命令式的 thunk 对于复杂异步数据流的表现力有限


先看名字来理解：saga，这个术语常用于CQRS架构，代表查询与责任分离。

没错，就是查询（dispatch）与责任（sagas）分离。saga提供了action监听函数，只需在组件里dispatch 相应type的action，就可以自动调用你定义好的对应这个action的异步处理函数（sagas）来完成任务，保证了只在组件里dispatch action来发起异步操作而不是redux-thunk、redux-promise的调用action creators。

另外一大特色就是redux-saga做到了异步代码以同步方式写，非常直观方便，怎么做到的呢？它是利用了ES6新魔法Generator迭代器，可以完美解决异步回调地狱，让你以同步方式写异步。saga正是利用Generator特性让其处理异步变得非常方便又容易理解。这是一个常见的请求后台数据的异步操作，感受一下：


## redux-thunk
[Redux官方文档-thunk](http://cn.redux.js.org/docs/advanced/AsyncActions.html)

> 通过使用指定的 middleware，action 创建函数除了返回 action 对象外还可以返回函数。这时，这个 action 创建函数就成为了 thunk

> 当 action 创建函数返回函数时，这个函数会被 Redux Thunk middleware 执行。这个函数并不需要保持纯净；它还可以带有副作用，包括执行异步 API 请求。这个函数还可以 dispatch action，就像 dispatch 前面定义的同步 action 一样
> thunk 的一个优点是它的结果可以再次被 dispatch

判断 actionCreator 返回的是不是一个函数，如果不是的话，就很普通地传给下一个中间件（或者 reducer）；如果是的话，那么把 dispatch、getState、extraArgument 作为参数传入这个函数里，实现异步控制
```js
function createThunkMiddleware(extraArgument) {
    return ({ dispatch, getState }) => next => action => {
        if (typeof action === 'function') {
            return action(dispatch, getState, extraArgument);
        }
        return next(action);
    };
}
```

getState可以在reducer里面获取store里面的数据
`const method = getState().cashier.method;`



举个比较简单的例子，我们现在要实现『图片上传』功能，用户点击开始上传之后，显示出加载效果，上传完毕之后，隐藏加载效果，并显示出预览图；如果发生错误，那么显示出错误信息，并且在2秒后消失。
###### redux-thunk
```js
function upload(data){
    return dispatch => {
    	// 显示出加载效果
    	dispatch({ type: 'SHOW_WAITING_MODAL' });
    	// 开始上传
    	api.upload(data)
    	    .then(res => {
    		// 成功，隐藏加载效果，并显示出预览图
	    	dispatch({ type: 'PRELOAD_IMAGES', data: res.images });
	    	dispatch({ type: 'HIDE_WAITING_MODAL' });
	    	})
	    .catch(err => {
	    	// 错误，隐藏加载效果，显示出错误信息，2秒后消失
	    	dispatch({ type: 'SHOW_ERROR', data: err });
	    	dispatch({ type: 'HIDE_WAITING_MODAL' });
	    	setTimeout(_ => dispatch({ type: 'HIDE_ERROR' }), 2000);
	    })
    }
}
```
###### redux-saga
```js
import { take, put, call, delay } from 'redux-saga/effects'
// 上传的异步流
function *uploadFlow(action) {
	// 显示出加载效果
  	yield put({ type: 'SHOW_WAITING_MODAL' });
  	// 简单的 try-catch
  	try{
  	    const response = yield call(api.upload, action.data);
	    yield put({ type: 'PRELOAD_IMAGES', data: response.images });
	    yield put({ type: 'HIDE_WAITING_MODAL' });
  	}catch(err){
  	    yield put({ type: 'SHOW_ERROR', data: err });
	    yield put({ type: 'HIDE_WAITING_MODAL' });
	    yield delay(2000);
	  	yield put({ type: 'HIDE_ERROR' });
  	} 	
}


function* watchUpload() {
  yield* takeEvery('BEGIN_REQUEST', uploadFlow)
}
```