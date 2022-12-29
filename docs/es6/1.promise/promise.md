# promise

## 一、promise解决什么样的问题

1. 我们写代码都是基于回调的，在串行模式当中，上一个函数的输出通常作为下一个函数的输入，这样就容易形成回掉地狱
2. 解决异步并发问题，promise.all
3. 错误处理变得更加容易

## 二、什么是promise

主要有三个状态，pending、reject、fullfield；只有在pending的状态才可以进行状态的改变

1. pending -> reject: 两种方式会使promise走向失败状态，reject和 throw Error
2. pending -> fulfield

**promise的执行器回掉函数是立即执行函数** 即如下

```js
const promise = new Promise((resolve, reject) => {
   console.log(1)
   resolve('ok')
}) 
console.log(2) // 输出顺序：1, 2
```

## 三、手写promise基础函数

1. 由于promise是存在3种状态的，所以定义三个常量用来标记promise的状态

```js
const PENDING = "PENDING";
const SUCCESS = "FULFILLED";
const FAIL = "REJECTED";
```

2. 创建promise的方式是使用`const promise = new Promise((resolve, reject) => {})`由此可见:

+ Promise实际上是一个构造函数，并且是以一个函数作为参数，此时我们采用class进行书写Promise类，方便对象的私有变量的书写
+ promise的执行器回掉函数（executor）是立即执行函数，接受两个函数作为参数，resolve函数代表成功状态的函数，reject函数代表失败状态函数且会有一个失败原因抛出，两个函数主要是对状态的改变的一个说明，并且通知订阅者状态发生了改变，由于两种方式会使promise走向失败状态，reject和 throw Error，所以executor是放在trycatch函数体当中执行
  
## 四、 详写promise的then方法

promise具备一个then函数，接受两个函数作为参数

- state的状态是成功状态调用onFulfilled函数，如果onFulfilled不是一个函数，需要被忽略
- state的状态是失败状态调用onRejected函数，如果onRejected不是一个函数，需要被忽略
- 如果promise内是一个异步的任务，那state的状态依旧是pending状态，此时需要将onFulfilled函数和onRejected函数分别存到成功函数队列和失败函数队列，当状态发生改变的时候一起执行并且一个promise当中可以多次调用then方法，所有的成功和失败的回掉函数都是会按照顺序进行执行

promise的then函数的返回值：

+ 1. 如果返回值data不是一个promise，那么此次then函数无论是reject还是resolve的返回值data都会被传递到 下一个then函数的成功回掉函数resolve当中作为resolve的入参
+ 2. 如果返回值data不是一个promise，但是在下一个then函数当中又想执行reject错误回掉函数，那么只有在本次的成功或者失败的回掉函数当中抛出异常`throw new Error()`才可以做到
+ 3. 如果返回的是一个pending的promise，那么回掉链会中断
+ 4. then可以无限调用，但是不能像JQuery那样通过`return this`来实现，因为promise不能从成功状态走向失败状态，而是需要根据每次的then的结果来决定下一个then的执行，所以我们在书写then方法的时候需要返回一个Promise的实例，通过Promise的返回值决定下一次then方法的走向
+ 5. excutor是被包裹在trycatch代码块当中执行的，所以会捕获最外层的异常，一旦promise内部存在异步任务且抛出异常的情况下，就不能够捕获到了，在then方法当中继续包裹trycatch代码块
+ 6. 如果返回值是一个promise，会去解析返回的promise的值，根据返回值的成功还是失败决定下一个then函数执行resolve还是reject，promise的规范当中规定了then方法当中的onFulfilled和onRejected函数必须异步执行且给出解决方案可以包裹在宏任务或者微任务当中执行，所以将所有的trycatch代码块包裹在setTimeout当中，并且提供一个解析返回的promise的值的预处理函数
      
        在预处理函数`function resolvePromise(promise2, x, resolve, reject)`当中：
        
        + 1. 如果promise2和x相等，也就是promise2返回值是一个promise，那么就会陷入无限循环，相当于自己等自己状态改变，永远等不到就会抛出异常，promise规范当中规定好使用`new TypeError`抛出异常
        + 2. 如果x是一个普通值，直接`resolve(x)`
        + 3. 如果x是对象或者是一个函数，并且具有then方法，为了避免错误取值所以包裹在trycatch代码块当中执行，如果then方法不是一个函数，那就直接`resolve(x)`，如果then方法是一个函数，那么就将x作为then的this指向，执行then方法，如果then方法当中返回的还是一个promise的实例，那么将递归这个解析函数直到返回的是一个resolve常量或者reject常量
        + 4. resolve或者reject存在多个的情况下，只能是成功resolve或者失败reject的一种情况，且是第一次执行的情况,此时需要一个标记的状态位来表示是否已经执行过resolve或者reject
+ 7. 值穿透问题：如果onFulfilled不是一个函数，需要被忽略，将上一个函数的返回值传递给下一个then函数，onRejected是一样的道理

### 面试过程产生的问题？

> 1. 为什么promise的then方法要返回一个 promise的实例？
> 答： 因为then方法实现了链式调用
> 2. 为什么不效仿jquery方法的return this来实现链式调用？
> 答：因为promise是具有状态的，且在promise的excutor的回掉函数当中可能会具有异步任务，且promise的状态都是从pending状态转变成成功状态或者失败状态且不可逆，所以此处返回一个promise的实例刚好，因为新的promise的实例具有新的状态转变
> 3. 为什么在新返回的promise的实例当中要不断的调用resolve函数？
> 答： 因为then方法在返回值不是一个promise的时候，除抛出异常外，都会走到下一个promise的resolve函数当中，所以我们需要将此次的返回值直接传递给resolve函数使得此次的promise走向成功状态
> 4. excutor是被包裹在trycatch代码块当中执行的，所以会捕获最外层的异常，一旦promise内部存在异步任务且抛出异常的情况下，就不能够捕获到了，怎么办？
> 答： 在then方法当中继续包裹trycatch代码块
> 5. `let promise2 = p.then(() => { then: function () {} })` 这是一个promise吗？
> 答： 是，有then有函数就是promise

## 五、promise的测试方法

promisea+ 当中提供了测试方法，会调用deferred方法获取到promise实例

通过npm install promises-aplus-tests -g 全局安装

promises-aplus-tests promise.js 来检查是否符合规范

所以我们需要提供一个延迟对象的方法

```js
Promise.deferred = function () {
    let dfd = {}
    dfd.promise = new Promise((resolve, reject) => {
        dfd.resolve = resolve
        dfd.reject = reject
    })
    return dfd
}
```
