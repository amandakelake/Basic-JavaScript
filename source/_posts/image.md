---
title: Promise深入理解与实例分析
---

# 异步（一）：Promise深入理解与实例分析
#Front-End/JS/基础

[深入理解Promise运行原理 - 掘金](https://juejin.im/post/5a5ea6f56fb9a01cbf385e62) 自己动手实现简易版promise
[转promises 很酷，但很多人并没有理解就在用了 | SHANG Blog](https://blog.xinshangshangxin.com/2015/09/29/promise-problem/#%E6%96%B0%E6%89%8B%E9%94%99%E8%AF%AFNo-1%EF%BC%9A%E5%9B%9E%E8%B0%83%E9%87%91%E5%AD%97%E5%A1%94)讲了很多例子和一些坑，建议先看完基础再看它
[写一个符合 Promises/A+ 规范并可配合 ES7 async/await 使用的 Promise](https://zhuanlan.zhihu.com/p/23312442)
[八段代码彻底掌握 Promise - 掘金](https://juejin.im/post/597724c26fb9a06bb75260e8)

基础定义和API方面，这里就不说了，请自行学习

前面的理论部分基于《你不知道的JS》中卷第二部分第三章，可以结合前人的一些博客认真理解一下。 后面的代码实例非常有助于理解，并且我都做了注释，有基础的同学可以跳过理论部分直接参阅。


## 回调的缺陷
1、顺序不确定性
回调表达异步流程是非线性、非顺序的 
2、可信任性
回调会受到控制反转的影响，因为回调暗中把控制权交给了第三方（通常是不受控制的第三方工具）来调用代码中的continuation

## 一、Promise本质
先直接在控制台打印看一下它是什么
[image:D997CFEF-CC73-402C-9D40-BA6CF2959566-701-00001CFD8CA2C1D9/BE73465C-243B-402D-B444-4093FB658777.png]
展开后可以看到 Promise构造器上定义了resolve和reject方法，then()方法定义在其原型上。

这就解释了为什么下面两种写法都可以了
```js
Promise.resolve().then(() => {
    ...
}) 
```
```js
let p = new Promise((resolve, reject) => {
    ...
    resolve(someValue)
})
p.then(() => {
    ...
})
```

### 二、从事件循环角度理解Promise
Promise 所说的异步执行，只是将 Promise 构造函数中 resolve，reject 方法和注册的 callback 转化为 eventLoop的 microtask/Promise Job，并放到 Event Loop 队列中等待执行，也就是 Javascript 单线程中的“异步执行”

> 根据规范，microtask 存在的意义是：在当前 task 执行完，准备进行 I/O，repaint，redraw 等原生操作之前，需要执行一些低延迟的异步操作，使得浏览器渲染和原生运算变得更加流畅。这里的低延迟异步操作就是 microtask。原生的 setTimeout 就算是将延迟设置为 0 也会有 4 ms 的延迟，会将一个完整的 task 放进队列延迟执行，而且每个 task 之间会进行渲染等原生操作。假如每执行一个异步操作都要重新生成一个 task，将提高宿主平台的负担和响应时间。所以，需要有一个概念，在进行下一个 task 之前，将当前 task 生成的低延迟的，与下一个 task 无关的异步操作执行完，这就是 microtask。

```js
new Promise((resolve) => {
  console.log('a')
  resolve('b')
  console.log('c')
}).then((data) => {
  console.log(data)
})

// a, c, b
```
构造函数中的输出执行是同步的，输出 a, 执行 resolve 函数，将 Promise 对象状态置为 resolved，输出 c。
同时注册这个 Promise 对象的回调 then 函数。整个脚本执行完，stack 清空。
event loop 检查到 stack 为空，再检查 microtask 队列中是否有任务，发现了 Promise 对象的 then 回调函数产生的 microtask，推入 stack，执行。输出 b，event loop的列队为空，stack 为空，脚本执行完毕。

## 三、从thenable看Promise

识别 Promise(或者行为类似于 Promise 的东西)就是定义某种称为 thenable 的东 西，将其定义为任何具有 then(..) 方法的对象和函数。我们认为，任何这样的值就是 Promise 一致的 thenable

根据一个值的形态(具有哪些属性)对这个值的类型做出一些假定。这种类型检查(type check)一般用术语鸭子类型(duck typing)来表示

```js
function checkThenable(p) {
  if (p !== null && ( typeof p === 'object' || typeof p === 'function') && typeof p.then === 'function') {
    // 假设这是一个thenable
    return true
  } else {
    // 不是thenable
    return false
  }
}
```

### 1、then()接收两个函数作为参数
第一个参数是Promise执行成功时的回调，第二个参数是Promise执行失败时的回调。两个函数只会有一个被调用，函数的返回值将被用作创建then返回的Promise对象。

1、return 一个同步的值 ，或者 undefined（当没有返回一个有效值时，默认返回undefined），then方法将返回一个resolved状态的Promise对象，Promise对象的值就是这个返回值。
2、return 另一个 Promise，then方法将根据这个Promise的状态和值创建一个新的Promise对象返回。
3、throw 一个同步异常，then方法将返回一个rejected状态的Promise,  值是该异常。

太啰嗦了，总结一下then()方法的看家本领
* 返回另一个promise；
* 返回一个同步值（或者undefined）
* 抛出一个同步错误。

### 2、Promise 实例化时传入的函数会立即执行，then(...) 中的回调需要异步延迟调用
`Promise/A+规范`中解释：实践中要确保onFulfilled 和 onRejected 方法异步执行，且应该在 then 方法被调用的那一轮事件循环之后的新执行栈中执行。这个事件队列可以采用宏任务 macro-task机制或微任务 micro-task机制来实现



## 四、Promise的异步处理

Promise的两个固有行为：
1、每次对 Promise 调用 then(..)，它都会创建并返回一个新的 Promise，我们可以将其 链接起来;
2、不管从 then(..) 调用的完成回调(第一个参数)返回的值是什么，它都会被自动设置 为被链接 Promise(第一点中的)的完成。

使 Promise 序列真正能够在每一步有异步能力的关键是：**Promise. resolve(..) 会直接返回接收到的真正 Promise，或展开接收到的 thenable 值，并在持续展 开 thenable 的同时递归地前进**。

尝试去理解一下下面这段代码
```js
let p = Promise.resolve(1);
p
  .then(v => {
    console.log(v);
    // 创建一个promise并返回
    return new Promise((resolve, reject) => {
      // 引入异步，一样正常工作
      setTimeout(() => {
        resolve(v * 2);
      }, 4);
    });
  })
  .then(v => {
    // 猜猜拿到了多少？
    console.log(v);
  });
```
会发现：不管我们想要多少个异步步 骤，每一步都能够根据需要等待下一步(或者不等!)

## 五、Promise的错误处理

一个错误/异常是基于每个Promise的，意味着在链条的任意一点捕获这些错误是可能的，而且这些捕获操作在那一点上将链条“重置”，使它回到正常的操作上来

```js
let p = new Promise((resolve, reject) => {
  reject('error')
});
let p2 = p.then(() => {
  // 永远到达不了这里
  console.log('这句话不会出现')
})
```

再看一段代码
```js
let p = Promise.resolve(1);
p.then((v) => {
  console.log(v * 2);
  foo();//这一步，underfined出错
  // 再也到不了这里了
  return Promise.resolve(3);
}).then((v) => {
  console.log('到不了这里',v)
},(err) => {
  console.log('错误来这了',err);
  return 4
}).then((v) => {
  console.log(v)
})
```
第 2 步出错后，第 3 步的拒绝处理函数会捕捉到这个错误。拒绝处理函数的返回值(这段 代码中是 3)，如果有的话，会用来完成交给下一个步骤(第 4 步)的 promise，这样，这 个链现在就回到了完成状态。
[image:ED6EDBCD-A097-4B38-BF29-4A92BE83F349-701-00001A0C86A4BC7E/18341983-1A2A-4A3A-9314-CB530047BF8A.png]

注意这句话，解释了为什么最后会出现4，这里要好好理解透彻
> 拒绝处理函数的返回值(这段代码中是 3)，如果有的话，会用来完成交给下一个步骤(第 4 步)的 promise

总结起来，Promise的步骤

• 调用 Promise 的 then(..) 会自动创建一个新的 Promise 从调用返回。
• 在完成或拒绝处理函数内部，如果返回一个值或抛出一个异常，新返回的可链接的)Promise 就相应地决议。
• 如果完成或拒绝处理函数返回一个 Promise，它将会被展开，这样一来，不管它的决议值是什么，都会成为当前 then(..) 返回的链接 Promise 的决议值。

另外，记住这条结论，对于理解后面的例子有帮助
**当使用then(resolveHandler, rejectHandler)，rejectHandler不会捕获在resolveHandler中抛出的错误。**
个人习惯是从不使用then方法的第二个参数，转而使用catch()方法
但后面的例子是为了更清晰的讲述promise，所以几乎都用了第二个参数

## 六、Promise的穿透
下面这段代码先自己想一下，再去控制台打印
```js
Promise.resolve(1).then(Promise.resolve(2)).then((v) => {
  console.log(v)
})

Promise.resolve(1).then(return Promise.resolve(2)).then((v) => {
  console.log(v)
})

Promise.resolve(1).then(null).then((v) => {
  console.log(v)
})

Promise.resolve(1).then(return 2).then((v) => {
  console.log(v)
})

Promise.resolve(1).then(() => {
  return 2
}).then((v) => {
  console.log(v)
})
```

答案是
```js
1；
Uncaught SyntaxError: Unexpected token return；
1
Uncaught SyntaxError: Unexpected token return；
2
```
当then()受**非函数的参数**时，会解释为then(null)，这就导致前一个Promise的结果穿透到下面一个Promise。

所以要提醒你自己：**永远给then()传递一个函数参数**。


## 七、Promise局限性
**1、顺序错误处理**
Promise 链中的错误很容易被 无意中默默忽略掉
**2、单一值**
Promise 只能有一个完成值或一个拒绝理由

## Promise性能
Promise 进行的动作要多一些，这自然意味着它也会稍慢一些
更多的工作，更多的保护。这些意味着 Promise 与不可信任的裸回调相比会更慢一些
Promise 使所有一切都成为异步的了，即有一些立即(同步)完 成的步骤仍然会延迟到任务的下一步。这意味着一个 Promise 任务序列可能 比完全通过回调连接的同样的任务序列运行得稍慢一点

Promise 稍慢一些，但是作为交换，你得到的是大量内建的可信任性、对 Zalgo 的避免以及 可组合性


## 八、几个不错的例子
### 1、理解三种状态
```
var p1 = new Promise(function(resolve,reject){
  resolve(1);
});
var p2 = new Promise(function(resolve,reject){
  setTimeout(function(){
    resolve(2);  
  }, 500);      
});
var p3 = new Promise(function(resolve,reject){
  setTimeout(function(){
    reject(3);  
  }, 500);      
});
// 直接返回1
console.log(p1);
// 由于加入了异步，而且是事件循环中的宏任务，所以暂时处于pending状态，underfined
console.log(p2);
// 同理，pending状态
console.log(p3);

// 直接加到下一个事件循环，暂时没输出，最后会输出resolve 2
setTimeout(function(){
  console.log(p2);
}, 1000);
// 同理，在下一个事件循环，最后会输出reject 3
setTimeout(function(){
  console.log(p3);
}, 1000);

// promise属于事件循环中的微任务，所以要比上两个setTimeout输出的快，1
p1.then(function(value){
  console.log(value);
});
// 同理，2
p2.then(function(value){
  console.log(value);
});
// 这里注意是catch,所以输出3
p3.catch(function(err){
  console.log(err);
});
```

[image:61135F25-223D-4518-904C-1F1048D8C4EE-701-00001B013CAA7644/AE67A4AF-54FB-478B-BA1B-6C73E55BD27C.png]

### 2、链式调用以及返回值
```
var p = new Promise(function(resolve, reject){
  resolve(1);
});
p.then(function(value){               //第一个then
  console.log(value); // 1
  return value*2;
}).then(function(value){              //第二个then
  console.log(value); // 2
}).then(function(value){              //第三个then
  console.log(value); // underfined
  return Promise.resolve('resolve'); 
}).then(function(value){              //第四个then
  console.log(value); // 'resolve'
  return Promise.reject('reject');
}).then(function(value){              //第五个then
  console.log('resolve: '+ value); // 不到这里，没有值
}, function(err){
  console.log('reject: ' + err);  // 'reject'
})
```

[image:494BB957-7D18-48CD-881A-B16C1FA10615-701-00001B29CC8C6750/7DA20400-34B6-4028-8196-62861AF21E99.png]

上面说的` then()接收两个函数作为参数`，`返回值有三种情况`，可以返回上面看看

### 3、异常处理
```
let p1 = new Promise((resolve, reject) => {
  foo();
  resolve(1)
})
p1.then((v) => {
  console.log('1不会到这里')
},(err) => {
  console.log('p1的第一次错误来了这里',err)
}).then((v) => {
  console.log('p1第二次，在这里拿到了underfined',v)
},(err) => {
  console.log('第二次，没有错误，这里不会出现',err)
})

let p2 = new Promise((resolve,reject) => {
  resolve(2);
})
p2.then((v) => {
  console.log('p2第一次的值2来这里了',2);
  foo()
},(err) => {
  console.log('p2这里不会拿到第一次的错误',err)
}).then((v) => {
  console.log('p2上面第一次有错误，这里不会有值',v)
},(err) => {
  console.log('这里拿到了p2上一次的错误',err);
  return '即使错误，也能继续传值'
}).then((v) => {
  console.log('到这里应该很清晰了吧',v)
},(err) => {
  console.log('这里已经没有错误了',err)
})
```

Promise中的异常由then参数中第二个回调函数（Promise执行失败的回调）处理，异常信息将作为Promise的值。异常一旦得到处理，then返回的后续Promise对象将恢复正常，并会被Promise执行成功的回调函数处理。

需要注意p1、p2 多级then的回调函数是交替执行的 ，这正是由Promise then回调的异步性决定的。

### 4、resolve与reject的区别
```
var p1 = new Promise(function(resolve, reject){
  resolve(Promise.resolve('resolve'));
});
p1.then(
  function fulfilled(value){
    console.log('fulfilled: ' + value);
  }, 
  function rejected(err){
    console.log('rejected: ' + err);
  }
);
```
这段毫无疑问，resolve直通车

```
var p2 = new Promise(function(resolve, reject){
  resolve(Promise.reject('reject'));
});
p2.then(
  function fulfilled(value){
    console.log('fulfilled: ' + value);
  }, 
  function rejected(err){
    console.log('rejected: ' + err);
  }
);
```
这段可能会有点疑问，主要在于理解这句代码`resolve(Promise.reject('reject'));`
再回想一下上面的错误处理以及thenable对象的展开功能，是不是就好理解一点了，其实可以理解为与运算（&&）,有一个reject，传下去的也会是reject

但是！！！并不是一直链式的传下去的全都是reject，只是紧跟着的下一个then会收到reject而已，万望好好理解这句话（我这里不展开讲了）

```
var p3 = new Promise(function(resolve, reject){
  reject(Promise.resolve('resolve'));
});
p3.then(
  function fulfilled(value){
    console.log('fulfilled: ' + value);
  }, 
  function rejected(err){
    console.log('rejected: ' + err);
  }
);
```
有了第二段的基础，这一段应该就非常好理解了


如果上述内容，看得不是很懂，建议多看几遍（不一定看我这篇，看看前人的也好），正所谓“读书百遍其义自见”