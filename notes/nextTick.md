# 异步更新机制

参考： ['learnVue'](https://github.com/answershuto/learnVue)  ['vue-analysis'](https://github.com/ustbhuangyi/vue-analysis)

首先来看一个小demo
```javascript
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

> 未完待续