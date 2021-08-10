# JavaScript异步编程

JavaScript异步编程的方式主要有四种：

- 回调函数
- `Promise`
- `async/await`异步函数
- `Generator`生成器

本文主要是想通过一个场景，对比四种方式的不同实现，更好的理解各自适用场景，解决的问题

## 场景

完成一个业务逻辑，需要调用三个接口，后面的接口调用分别依赖上个接口的返回

## 实现

### 回调函数的方式

使用回调的方式，代码实现如下，可以看到代码结构会有三层嵌套，如果逻辑再复杂些，嵌套就会更多，以至于出现所谓的`callback hell`

```js
let [api0, api1, api2] = Array(3).fill(
  (data, callback) => {
    setTimeout(() => callback(data + 1))
  }
)
api0(0, ret0 => {
  api1(ret0, ret1 => {
    api2(ret1, ret2 => {
      console.log(ret2) // 输出3
    })
  })
})
```

### Promise的方式

为了解决`callback hell`的问题，ES6推出了`Promise`语法

```js
let [api0, api1, api2] = Array(3).fill(
  params => new Promise(resolve => resolve(params + 1))
)
api0(0)
  .then(ret0 => api1(ret0))
  .then(ret1 => api2(ret1))
  .then(ret2 => console.log(ret2)) // 输出3
```

使用`Promise`的方式，代码结构减少到一层嵌套，逻辑也更加清晰

另外，如果一个异步请求，依赖于其它多个异步请求的结果，在这种场景下，对比使用回调的实现，使用`Promise`，在代码结构上的简化就更加明显了：

```js
// 回调的实现
let count = 0
let ret0
let ret1
api0(0, _ret0 => {
  ret0 = _ret0
  ++count === 2 && api2(ret0, ret1)
})
api1(0, _ret1 => {
  ret1 = _ret1
  ++count === 2 && api2(ret0, ret1)
})
```

```js
// Promise的实现
Promise.all([
  api0(0),
  api1(0)
]).then(
  (ret0, ret1) => api2(ret0, ret1)
)
```

### async/await异步函数的方式

ES8基于`Promise`引入异步函数，让异步的代码看起来就像同步代码一样，异步函数的目的是为了优化对`Promise`的同步使用（即，一个`Promise`的调用依赖于另一个`Promise`的完成，也就是我们当前讨论的场景）

```js
let [api0, api1, api2] = Array(3).fill(
  params => new Promise(resolve => resolve(params + 1))
)
// await关键词只能在异步函数中使用，所以这里需要声明一个异步函数来封装代码
async function asyncWrap() {
  let ret0 = await api0(0) 
  let ret1 = await api1(ret0)
  let ret2 = await api2(ret1)
  console.log(ret2)
}
asyncWrap()
```

### Generator生成器的方式

与异步函数`async/await`结构相近的另外一种异步实现就是生成器`Generator`，不同的是，生成器的异步执行需要手动触发

为了使用上生成器，并让生成器能够自动执行，我们首先需要将函数`thunk`化。所谓的`thunk`，就是将函数柯里化为`callback`作为唯一或第一参数的函数，即：

```js
fn(...args, callback) => trunkifyFn(...args)(callback)
```

[thunkify](https://github.com/tj/node-thunkify)函数的简易版实现：

```js
function thunkify(fn) {
  return function(...args) {
    return callback => {
      fn.call(this, ...args, callback)
    }
  }
}
```

`trunkify`只是对代码结构做了调整，这种调整是为了配合流程控制器[co](https://github.com/tj/co)实现生成器函数的自动执行，`co`函数的简易版实现：

```js
function co(gen){
    let g = gen()
    function next(...args){
        let { value: thunkifyFn } = g.next(...args)
        thunkifyFn && thunkifyFn(...ret => next(...ret))
    }
    next()
}
```

有了`thunkify`和`co`两个工具之后，就可以实现上述场景逻辑了：

```js
let [api0, api1, api2] = Array(3).fill(
  thunkify((params, callback) => {
    setTimeout(() => callback(params + 1))
  })
)
function* genWrap() {
  let ret0 = yield api0(0)
  let ret1 = yield api1(ret0)
  let ret2 = yield api2(ret1)
  console.log(ret2)
}
co(genWrap)
```

相比`Promise`和`async/await`的实现，生成器的实现多了更多逻辑，理解起来也更复杂些，显然生成器并不适合于该场景

生成器适用于组装流程，比如，在Koa框架中，用生成器来实现中间件：

![img](https://camo.githubusercontent.com/d80cf3b511ef4898bcde9a464de491fa15a50d06/68747470733a2f2f7261772e6769746875622e636f6d2f66656e676d6b322f6b6f612d67756964652f6d61737465722f6f6e696f6e2e706e67)

一个简易版本的实现：

```js
function App() {
  this.middlewares = []
}
App.prototype.use = function(mw) {
  this.middlewares.push(mw)
}
App.prototype.run = function() {
  const mws = this.middlewares
  let mw
  for(let i = mws.length - 1; i >= 0; i--) {
    mw = mws[i](mw && mw.next.bind(mw))
  }
  function _run(next) {
    let {value} = next()
    value && _run(value)
    next()
  }
  _run(mw.next.bind(mw))
}

const app = new App()

app.use(function* (next) {
  console.log(0)
  yield next
  console.log(1)
})
app.use(function* (next) {
  console.log(2)
  yield next
  console.log(3)
})
app.use(function* (next) {
  console.log(4)
  yield next
  console.log(5)
})
app.run() // 输出 0 2 4 5 3 1
```

## 总结

- 回调函数比较容易理解，实现简单，但是会导致`callback hell`问题
- `Promise`能很好处理`callback hell`问题，也是`async/await`异步函数的实现基础
- `async/await`异步函数使异步代码的结构接近同步代码，更加简洁，可读性更高，用于优化异步函数的同步调用
- `Generator`生成器的调用者可以控制生成器的执行和暂停