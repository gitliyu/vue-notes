# 异步更新机制

首先来看一个小demo
```html
<template>
  <div>
    <div ref="num">{{ num }}</div>
  </div>
</template>

<script type="text/javascript">
  export default {
    data () {
      return {
        num: 1
      }
    },
    mounted () {
      this.num = 2;
      console.log(this.$refs.num.innerHTML);
      this.$nextTick(() => {
        console.log(this.$refs.num.innerHTML);
      });
    }
  }
</script>
```
在数据修改完成后打印出dom中的内容，控制台的结果是
```
1
2
```
说明在数据更新之后，并没有立刻对dom进行更新

之前在['Observer与响应式数据'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-observer.md)对`Watcher`进行介绍时，我们了解到在响应式数据进行更新时，会将`Watcher`放在了一个异步更新队列中，在下一次dom更新即`nextTick`的时候，对队列中所有`Watcher`进行更新，所以我们需要想要拿到更新后的dom，就需要在`nextTick`完成后再进行操作

官网对这部分是这样描述的
> 可能你还没有注意到，`Vue`异步执行`DOM`更新。只要观察到数据变化，`Vue` 将开启一个队列，并缓冲在同一事件循环中发生的所有数据改变。如果同一个`watcher` 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和`DOM` 操作上非常重要。然后，在下一个的事件循环“tick”中，`Vue`刷新队列并执行实际 (已去重的) 工作。`Vue` 在内部尝试对异步队列使用原生的`Promise.then`和`MessageChannel`，如果执行环境不支持，会采用`setTimeout(fn, 0)`代替

### 异步更新队列
之前有提到，响应式数据变化时会触发`Watcher`的`update`方法，位于`src/core/observer/watcher.js`
```javascript
update () {
  if (this.computed) {
    // 计算属性相关
    if (this.dep.subs.length === 0) {
      // 不希望执行计算，标记dirty
      this.dirty = true
    } else {
      this.getAndInvoke(() => {
        this.dep.notify()
      })
    }
  } else if (this.sync) {
    // 同步时直接渲染视图
    this.run()
  } else {
    // 异步时加入Watcher队列
    queueWatcher(this)
  }
}
```
可以看到，对于未标记位同步更新的数据，都会调用`queueWatcher`方法加入到异步更新队列中，方法位于`src/core/observer/scheduler.js`

```javascript
export function queueWatcher (watcher: Watcher) {
  // 获取watcher的id
  const id = watcher.id
  // 检验id是否存在，已经存在则直接跳过，不存在则标记has，用于下次检验
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      // 如果未被激活，添加到队列中
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i >= 0 && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(Math.max(i, index) + 1, 0, watcher)
    }
    // 非等待状态下激活队列
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```
`queueWatcher`的作用是将一个观察者对象push进观察者队列，在判断`waiting`标识符后调用`nextTick`方法对队列进行更新

### nextTick
Vue用了一个单独的文件来定义`nextTick`方法，并且在生命周期过程中将其注册在了`Vue.nextTick`和`vm.$nextTick`上，代码位于`src/core/util/next-tick.js`
```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 回调函数加入callbacks
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  // 选择异步更新方式
  if (!pending) {
    pending = true
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // 返回一个Promise
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```
`nextTick`方法本身的逻辑非常简单，可以分为3部分
1. 把传入的回调函数`cb`push进`callbacks`数组
2. 根据`useMacroTask`标识符判断，选择异步更新方法`macroTimerFunc`或者`microTimerFunc`
3. 在没有传入回调函数的情况下返回一个`Promise`，所以其实`nextTick`还有以下写法
```javascript
this.$nextTick().then(() => {
  this.callback();
})
```

关于异步更新方法的定义
```javascript
let microTimerFunc
let macroTimerFunc
let useMacroTask = false

// 定义macroTimerFunc
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // 使用setImmediate
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  // 使用MessageChannel
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  // 使用setTimeout
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

// 定义microTimerFunc
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  // 使用Promise
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
} else {
  // 走macroTimerFunc流程
  microTimerFunc = macroTimerFunc
}
```
在`next-tick.js`文件中声明了`microTimerFunc`和`macroTimerFunc`2个变量，并对其进行定义

首先是`macroTimerFunc`，优先检测是否支持原生['setImmediate'](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/setImmediate)，这是一个高版本`IE`和`Edge` 才支持的特性，不支持的话再去检测是否支持原生的['MessageChannel'](https://developer.mozilla.org/zh-CN/docs/Web/API/MessageChannel)，即`postMessage`和`onMessage`方法，如果也不支持的话就会使用`setTimeout`

而对于`macroTimerFunc`的定义，会检测浏览器是否原生支持`Promise`，不支持的话直接指向`macroTimerFunc`的实现。

```javascript
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```
`flushCallbacks`方法会从`callbacks`队列中拿到回调函数并执行，接下来看一下回调函数`flushSchedulerQueue`的实现

### flushSchedulerQueue
```javascript
function flushSchedulerQueue () {
  // 表示为已激活
  flushing = true
  let watcher, id

  // 给queue排序，保证watchers执行顺序
  queue.sort((a, b) => a.id - b.id)

  // 这里不用index = queue.length;index > 0; index--的方式写
  // 因为在执行处理现有watcher对象期间，更多的watcher对象可能会被push进queue
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    id = watcher.id
    // 将has的标记删除
    has[id] = null
    // 执行watcher的run方法更新视图
    watcher.run()
    // 在测试环境中，检测watch是否在死循环中
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // 得到队列的拷贝
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  // 重置队列所有标识符的状态
  resetSchedulerState()

  // 调用activated钩子
  callActivatedHooks(activatedQueue)
  // 调用updated钩子
  callUpdateHooks(updatedQueue)

  // devtool hook
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```
主要功能是执行了队列中所有`Watcher`的`run`方法来更新视图，并在完成后出发了`updated`钩子函数